# 第4章：Init进程与系统启动

Init进程是Android系统中第一个用户空间进程，承担着整个系统的初始化重任。本章将深入剖析Init进程的实现原理，包括其启动流程、RC脚本解析、属性服务、SELinux策略加载等核心机制。通过与Linux传统init、iOS的launchd、鸿蒙OS的init对比，读者将全面理解Android独特的系统启动架构。

## 4.1 Init进程源码分析

### 4.1.1 Init进程的诞生

当Linux内核完成自身初始化后，会启动第一个用户空间进程——Init进程（PID=1）。Android的Init进程实现位于`system/core/init/`目录，其入口函数是`main()`。内核通过`run_init_process()`调用`/init`二进制文件，这是ramdisk中的第一个程序。

**内核到用户空间的转换**
```
kernel_init() -> run_init_process("/init") -> execve("/init")
```

Init进程的生命周期分为两个阶段：

**First Stage Init**
- 挂载基础文件系统（/dev、/proc、/sys等）
  - 使用`mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755")`
  - 创建必要的子目录：/dev/pts、/dev/socket、/dev/dm-user
- 创建设备节点
  - `/dev/null`、`/dev/zero`、`/dev/full`等基础设备
  - `/dev/kmsg`用于内核日志
  - `/dev/random`、`/dev/urandom`用于随机数
- 初始化内核日志
  - 通过`android::base::InitLogging()`设置日志系统
  - 重定向stdout/stderr到`/dev/kmsg`
- 加载SELinux策略（如果是强制模式）
  - 调用`SelinuxInitialize()`加载编译后的策略
  - 设置进程上下文为`u:r:init:s0`
- 执行Second Stage Init
  - 通过`execv("/system/bin/init", argv)`重新执行自身
  - 传递`--second-stage`参数标识

**Second Stage Init**
- 初始化属性服务（Property Service）
  - 创建属性共享内存区域
  - 启动Unix域套接字监听属性设置请求
  - 加载默认属性文件（default.prop、build.prop等）
- 解析RC脚本
  - 从`/system/etc/init/`、`/vendor/etc/init/`等目录加载
  - 构建Action和Service的内部数据结构
  - 验证语法正确性和权限要求
- 启动系统服务
  - 根据class和触发器有序启动
  - 设置进程的capabilities、优先级、cgroup等
- 进入主事件循环
  - 基于epoll的事件驱动模型
  - 处理属性变化、子进程退出、控制消息等

**关键环境变量设置**
```
PATH=/system/bin:/system/xbin:/vendor/bin
LD_LIBRARY_PATH=/system/lib64:/vendor/lib64
ANDROID_ROOT=/system
ANDROID_DATA=/data
```

### 4.1.2 关键数据结构

Init进程使用三个核心数据结构管理系统启动：

**Action（动作）**
```cpp
struct Action {
    std::string name;
    std::vector<Command> commands;
    std::map<std::string, std::string> property_triggers;
    std::string event_trigger;
    bool oneshot;
    int trigger_order;  // 触发优先级
    
    // 执行状态管理
    size_t current_command;
    bool ExecuteOneCommand();
    bool CheckPropertyTriggers();
}
```
Action代表一组命令的集合，可以被触发器（Trigger）激活执行。每个Action可以包含多个命令，按顺序执行。

**Service（服务）**
```cpp
struct Service {
    std::string name;
    unsigned flags;  // SVC_RUNNING, SVC_DISABLED等
    pid_t pid;
    std::vector<std::string> args;
    std::vector<Option> options;
    
    // 服务控制
    int crash_count;
    time_t time_started;
    time_t time_crashed;
    
    // 资源限制
    std::vector<std::pair<int, rlimit>> rlimits;
    std::vector<std::string> writepid_files;
    
    // 安全上下文
    std::string seclabel;
    std::vector<uid_t> supp_gids;
    CapSet capabilities;
    
    // 命名空间隔离
    unsigned namespace_flags;  // CLONE_NEWPID, CLONE_NEWNET等
}
```
Service代表需要启动和管理的守护进程，包含完整的进程管理信息。

**Property（属性）**
```cpp
// 属性存储结构
struct prop_info {
    atomic_uint_least32_t serial;  // 版本号，用于检测变化
    char value[PROP_VALUE_MAX];   // 属性值（92字节）
    char name[PROP_NAME_MAX];     // 属性名（32字节）
};

// 属性区域管理
struct prop_area {
    uint32_t bytes_used;
    atomic_uint_least32_t serial;
    uint32_t magic;
    uint32_t version;
    char data[0];  // 实际属性数据
};
```
系统属性采用共享内存实现，通过`property_service.cpp`管理，提供跨进程的配置共享机制。属性存储在多个内存区域中，按前缀分组以提高查找效率。

**命令管理器（CommandManager）**
```cpp
class BuiltinFunctionMap {
    std::map<std::string, BuiltinFunction> functions_;
    
    // 注册的内置命令
    void RegisterBuiltinFunctions() {
        Register("chmod", do_chmod);
        Register("chown", do_chown);
        Register("class_start", do_class_start);
        Register("copy", do_copy);
        Register("exec", do_exec);
        Register("mkdir", do_mkdir);
        Register("mount", do_mount);
        Register("setprop", do_setprop);
        Register("start", do_start);
        Register("stop", do_stop);
        Register("trigger", do_trigger);
        Register("write", do_write);
        // ... 更多命令
    }
};
```

### 4.1.3 事件循环机制

Init进程的主循环基于epoll实现，监听以下事件：
- 属性设置请求（通过Unix域套接字）
- 子进程退出信号（SIGCHLD）
- 其他进程的控制命令
- 内核消息（通过netlink套接字）

**Epoll事件源注册**
```cpp
void Epoll::Open() {
    epoll_fd_ = epoll_create1(EPOLL_CLOEXEC);
    
    // 注册信号处理
    RegisterHandler(signal_fd, HandleSignals);
    
    // 注册属性服务
    RegisterHandler(property_fd, HandlePropertySet);
    
    // 注册控制消息
    RegisterHandler(init_fd, HandleInitSocket);
    
    // 注册子进程监控
    RegisterHandler(sigchld_fd, HandleSigchld);
}
```

**主循环实现**
```cpp
int main() {
    // ... 初始化代码 ...
    
    while (true) {
        // 1. 执行待处理的Actions
        if (!ActionQueue.empty()) {
            auto action = ActionQueue.front();
            action->ExecuteOneCommand();
            if (action->IsCompleted()) {
                ActionQueue.pop();
            }
        }
        
        // 2. 检查需要重启的服务
        for (auto& service : ServiceList) {
            if (service.flags & SVC_RESTARTING) {
                if (CurrentTime() > service.time_crashed + 
                    service.restart_period) {
                    service.Start();
                }
            }
        }
        
        // 3. 计算超时时间
        int timeout = -1;  // 默认无限等待
        if (!ActionQueue.empty()) {
            timeout = 0;  // 有待处理命令，立即返回
        } else if (HasRestartingServices()) {
            timeout = CalculateRestartTimeout();
        }
        
        // 4. 等待事件
        epoll_event events[32];
        int nr = epoll_wait(epoll_fd, events, 32, timeout);
        
        // 5. 处理事件
        for (int i = 0; i < nr; i++) {
            auto handler = handlers_[events[i].data.fd];
            handler();
        }
    }
}
```

**信号处理机制**
```cpp
void HandleSignals() {
    signalfd_siginfo info;
    read(signal_fd, &info, sizeof(info));
    
    switch (info.ssi_signo) {
        case SIGCHLD:
            ReapChildren();
            break;
        case SIGTERM:
        case SIGINT:
            HandleShutdown();
            break;
        case SIGUSR1:
            HandleBootchart();
            break;
    }
}

void ReapChildren() {
    while (true) {
        int status;
        pid_t pid = waitpid(-1, &status, WNOHANG);
        if (pid <= 0) break;
        
        Service* service = FindServiceByPid(pid);
        if (service) {
            service->Reap(status);
            if (service->flags & SVC_CRITICAL) {
                // 关键服务崩溃，可能触发重启
                HandleCriticalCrash(service);
            }
        }
    }
}
```

### 4.1.4 与其他系统的对比

**Linux传统init（SysV/systemd）**

*SysV init特点：*
- 基于运行级别（0-6），每个级别对应不同的系统状态
- 使用Shell脚本（/etc/init.d/），串行执行
- 简单但启动速度慢，缺乏依赖管理
- 通过`update-rc.d`或`chkconfig`管理服务

*systemd特点：*
- 基于Unit文件（.service、.socket、.target等）
- 支持依赖关系（Requires、After、Before等）
- 并行启动服务，显著提升启动速度
- Socket激活和D-Bus激活
- 内置日志系统（journald）
- Cgroup集成，更好的资源管理

*Android init对比：*
- 更轻量级，专为嵌入式设备优化
- RC脚本比systemd unit更简单
- 触发器机制更灵活，适合动态系统状态
- 深度集成Android特性（属性系统、SELinux）

**iOS launchd**

*架构特点：*
- 统一的进程管理器，替代了init、inetd、cron等
- 使用XML plist文件定义服务
- 支持多种启动条件（RunAtLoad、StartInterval、WatchPaths等）
- XPC（跨进程通信）原生支持

*与Android init对比：*
```
iOS launchd plist:
<dict>
    <key>Label</key>
    <string>com.apple.service</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/service</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>

Android RC:
service apple_service /system/bin/service
    class core
    user system
    group system
```

*权限模型差异：*
- iOS：基于entitlements和沙箱profile
- Android：基于Linux UID/GID和SELinux
- iOS更封闭但更安全，Android更灵活

**鸿蒙OS init**

*分布式特性：*
- 服务可以跨设备启动和迁移
- 分布式软总线支持
- 能力（Ability）框架集成
- 支持原子化服务

*与Android对比：*
- 鸿蒙：`distributed_service`标签支持跨设备
- Android：仅支持本地服务管理
- 鸿蒙：内置分布式安全认证
- Android：需要额外实现跨设备通信

*配置文件对比：*
```json
// 鸿蒙服务配置
{
    "services": [{
        "name": "distributed_service",
        "path": "/system/bin/d_service",
        "distributed": true,
        "capability": ["ohos.permission.DISTRIBUTED_DATA"],
        "devices": ["phone", "tablet", "tv"]
    }]
}
```

## 4.2 RC脚本与系统属性

### 4.2.1 RC脚本语法

Android RC（Run Commands）脚本采用自定义的领域特定语言（DSL），主要包含四种语句类型：

**Action定义**
```rc
on <trigger> [&& <trigger>]*
    <command>
    <command>
    ...
```

**完整的Action示例**
```rc
# 早期初始化
on early-init
    start ueventd
    mkdir /mnt/vendor/persist 0771 root system
    
# 属性触发器
on property:sys.boot_completed=1
    setprop sys.runtime.firstboot.end ${ro.runtime.firstboot.end}
    exec_background - system system -- /system/bin/bootstat --record_boot_complete
    
# 多条件触发器
on property:vold.decrypt=trigger_restart_framework && property:ro.crypto.type=file
    class_start main
    class_start late_start
```

**Service定义**
```rc
service <name> <pathname> [ <argument> ]*
    <option>
    <option>
    ...
```

**完整的Service示例**
```rc
# 基础服务示例
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart apexd
    onrestart restart audioserver
    onrestart restart gatekeeperd
    onrestart class_restart main
    writepid /dev/cpuset/system-background/tasks
    
# 厂商服务示例（小米）
service mi_thermald /system/bin/mi_thermald
    class main
    user system
    group system
    capabilities SYS_NICE NET_ADMIN
    socket mi_thermald stream 0660 system system
```

**Import语句**
```rc
# 静态导入
import /init.${ro.hardware}.rc
import /vendor/etc/init/hw/init.${ro.hardware}.rc
import /system/etc/init/hw/init.${ro.zygote}.rc

# 动态导入（基于属性）
import /init.recovery.${ro.hardware}.rc
```

**触发器类型详解**

*启动阶段触发器：*
- `early-init`：最早的初始化阶段，SELinux尚未加载
- `init`：基础文件系统已挂载
- `late-init`：所有init.*触发器完成后
- `boot`：/data分区已挂载，加密状态确定
- `post-fs`：/system挂载后立即执行
- `post-fs-data`：/data挂载并解密后执行
- `zygote-start`：在启动zygote前执行
- `early-boot`：boot完成后的早期阶段
- `charger`：充电模式下的特殊触发器

*属性触发器：*
```rc
# 精确匹配
on property:persist.sys.language=zh-CN
    setprop persist.sys.locale zh-CN

# 通配符匹配
on property:sys.boot_from_charger_mode=1
    trigger late-init

# 多属性组合
on property:ro.crypto.state=encrypted && property:ro.crypto.type=file
    start vold_decrypt
```

*自定义触发器：*
```rc
# 定义触发器
on enable-logging
    start logd
    start logd-reinit

# 触发自定义触发器
on boot
    trigger enable-logging
```

### 4.2.2 RC脚本解析器

Init进程使用`ActionParser`、`ServiceParser`等类解析RC文件：

**解析器架构**
```cpp
class Parser {
    std::map<std::string, std::unique_ptr<SectionParser>> parsers_;
    
    void AddSectionParser(const std::string& name, 
                         std::unique_ptr<SectionParser> parser) {
        parsers_[name] = std::move(parser);
    }
    
    void ParseData(const std::string& data) {
        Tokenizer tokenizer(data);
        while (tokenizer.HasMore()) {
            auto tokens = tokenizer.GetLine();
            if (IsSectionStart(tokens[0])) {
                current_parser_ = parsers_[tokens[0]];
            }
            current_parser_->ParseLine(tokens);
        }
    }
};
```

**解析流程**

1. **词法分析（Tokenizer）**
```cpp
// 将文本分解为Token，处理引号、转义字符
std::vector<std::string> Tokenizer::GetLine() {
    // 跳过空白和注释
    SkipWhitespace();
    if (current_ == '#') SkipToNextLine();
    
    // 解析token
    while (!IsEndOfLine()) {
        if (current_ == '"') {
            tokens.push_back(ParseQuotedString());
        } else {
            tokens.push_back(ParseWord());
        }
    }
}
```

2. **语法分析（Parser）**
```cpp
class ActionParser : public SectionParser {
    void ParseSection(std::vector<std::string>& args) {
        // "on" <trigger> [&& <trigger>]*
        auto action = std::make_unique<Action>();
        for (size_t i = 1; i < args.size(); i++) {
            if (args[i] == "&&") continue;
            action->AddTrigger(args[i]);
        }
    }
    
    void ParseLine(std::vector<std::string>& args) {
        // 解析命令
        auto cmd = Command(args[0], args.begin() + 1, args.end());
        current_action_->AddCommand(cmd);
    }
};
```

3. **语义检查**
```cpp
bool Service::ParseLine(const std::vector<std::string>& args) {
    static const OptionParserMap option_parsers = {
        {"class", ParseClass},
        {"user", ParseUser},
        {"group", ParseGroup},
        {"capabilities", ParseCapabilities},
        {"seclabel", ParseSeclabel},
        {"oneshot", ParseOneshot},
        {"disabled", ParseDisabled},
        // ... 更多选项
    };
    
    auto parser = option_parsers.find(args[0]);
    if (parser == option_parsers.end()) {
        return Error("Invalid option: " + args[0]);
    }
    return parser->second(args);
}
```

4. **变量替换和条件处理**
```cpp
// 属性值替换
std::string ExpandProps(const std::string& src) {
    // ${property_name} -> property_value
    // ${property_name:-default} -> 带默认值
    std::regex prop_regex(R"(\$\{([^}]+)\})");
    return std::regex_replace(src, prop_regex, 
        [](const std::smatch& match) {
            return GetProperty(match[1].str());
        });
}

// 条件导入
if (android::base::GetBoolProperty("ro.debuggable", false)) {
    ParseConfig("/system/etc/init/debug.rc");
}
```

**解析优化**
- 预编译的二进制格式（.rcb文件）
- 缓存解析结果避免重复解析
- 并行解析多个RC文件
- 增量解析支持动态加载

### 4.2.3 系统属性服务

Android的Property Service提供了一个全局的键值对存储系统：

**属性分类**
- `ro.*`：只读属性，启动后不可修改
- `persist.*`：持久化属性，重启后保留
- `sys.*`：系统控制属性
- `debug.*`：调试属性

**实现机制**
1. 属性存储在共享内存区域（`/dev/__properties__`）
2. 通过Unix域套接字`/dev/socket/property_service`接收设置请求
3. Init进程验证权限后更新共享内存
4. 通过`property_changed`触发器通知监听者

**访问控制**
- SELinux上下文检查
- UID/GID权限验证
- 属性前缀白名单

### 4.2.4 厂商定制扩展

各厂商在RC脚本中添加了大量定制：

**小米MIUI**
- 添加`on property:ro.miui.version`触发器
- 自定义`mi_thermald`温控服务
- 扩展`persist.sys.miui.*`属性族

**OPPO ColorOS**
- `oplus_init`阶段用于厂商服务初始化
- 自定义`oplusreserve`分区挂载
- AI调度相关属性配置

**华为EMUI**
- `hw_init`触发器族
- 分布式能力相关服务配置
- 安全增强属性设置

## 4.3 SELinux策略加载

### 4.3.1 Android中的SELinux

SELinux（Security-Enhanced Linux）为Android提供了强制访问控制（MAC）机制。与传统Linux发行版不同，Android对SELinux进行了深度定制：

**Android SELinux特点**
- 精简的策略集，专注于应用隔离
- 与UID/GID权限模型协同工作
- 支持动态策略更新（部分场景）
- 厂商可扩展的策略框架

**策略组成**
1. **平台策略**（Platform Policy）：AOSP提供的基础策略
2. **厂商策略**（Vendor Policy）：设备制造商添加的策略
3. **ODM策略**（ODM Policy）：设备定制商的策略

### 4.3.2 策略加载流程

Init进程负责在启动早期加载SELinux策略：

**First Stage Init中的加载**
```
1. 检查内核是否支持SELinux
2. 从/system、/vendor等分区读取策略文件
3. 编译并加载到内核
4. 设置enforcing模式
```

**关键函数调用链**
- `SelinuxInitialize()` → `LoadPolicy()` → `security_load_policy()`
- 策略文件位置：`/system/etc/selinux/`, `/vendor/etc/selinux/`

**策略编译过程**
1. 读取多个CIL（Common Intermediate Language）文件
2. 使用`libsepol`合并策略
3. 编译为二进制格式
4. 通过`/sys/fs/selinux/load`加载到内核

### 4.3.3 上下文管理

SELinux上下文格式：`user:role:type:sensitivity[:categories]`

**Init进程的上下文转换**
- 初始上下文：`u:r:kernel:s0`
- First Stage：`u:r:init:s0`
- Second Stage：根据服务定义切换

**文件上下文恢复**
Init进程使用`restorecon`恢复文件的SELinux标签：
- `/file_contexts`定义文件标签规则
- 递归恢复关键目录（/data、/cache等）
- 支持增量恢复优化性能

### 4.3.4 与其他安全机制对比

**iOS沙箱**
- 基于BSD的MAC框架
- 以App为中心的隔离模型
- 更严格的进程间通信限制
- 无需显式策略配置

**鸿蒙安全架构**
- 分布式安全能力认证
- 基于标签的访问控制
- 跨设备安全策略同步
- AI驱动的动态权限调整

**Linux AppArmor**
- 路径基础的访问控制
- 更简单的策略语法
- 较低的性能开销
- 更适合桌面环境

## 4.4 早期启动优化技术

### 4.4.1 并行化启动

Android通过多种技术实现服务的并行启动：

**异步服务启动**
- 使用`class_start`批量启动同类服务
- 非关键服务延迟到`late_init`阶段
- 利用多核CPU并行初始化

**依赖管理优化**
```
service critical_service /system/bin/critical
    class core
    priority -20
    
service normal_service /system/bin/normal
    class late_start
    after critical_service
```

### 4.4.2 关键路径优化

**I/O优化**
- 预读取（readahead）关键文件
- 合并小文件读取请求
- 使用`mmap`减少内存拷贝

**内存优化**
- 延迟非必要的内存分配
- 共享只读数据段
- 优化RC脚本解析缓存

**CPU调度优化**
- 关键服务使用实时调度策略
- CPU亲和性设置
- 大小核调度优化

### 4.4.3 厂商优化案例

**高通Quick Boot**
- 保存内存快照到UFS/eMMC
- 跳过部分硬件初始化
- 缩短到2-3秒冷启动

**联发科Fast Boot**
- 硬件级启动加速
- 并行化外设初始化
- 预编译RC脚本缓存

**三星优化**
- 自定义`samsung_init`阶段
- AI预测服务启动顺序
- 动态调整启动优先级

### 4.4.4 启动时间分析工具

**bootchart**
- 记录启动过程的系统状态
- 生成可视化时间线图
- 识别启动瓶颈

**systrace**
- 更细粒度的性能分析
- 支持自定义追踪点
- 与Chrome追踪工具集成

**厂商工具**
- MIUI：`miui_boottime_analyzer`
- ColorOS：`oplus_boot_tracker`
- EMUI：`hw_bootprof`

## 本章小结

本章深入剖析了Android Init进程的核心机制：

1. **Init进程架构**：两阶段启动模型，事件驱动的主循环，与Linux/iOS/鸿蒙的设计对比
2. **RC脚本系统**：DSL语法设计，触发器机制，解析器实现，厂商扩展方案
3. **属性服务**：共享内存实现，权限控制模型，属性分类与持久化
4. **SELinux集成**：策略加载流程，上下文管理，与其他MAC机制对比
5. **启动优化**：并行化技术，I/O/内存/CPU优化，厂商定制方案，性能分析工具

关键要点：
- Init进程的事件驱动架构使得Android启动更加灵活可控
- RC脚本的触发器机制支持复杂的启动依赖管理
- 属性系统提供了轻量级的跨进程配置共享
- SELinux的早期加载确保了系统安全性
- 各厂商通过深度优化将启动时间压缩到秒级

## 练习题

### 基础题

**1. Init进程的PID为什么必须是1？**
<details>
<summary>提示：考虑孤儿进程的处理机制</summary>

答案：在Unix/Linux系统中，PID 1的进程有特殊职责：
- 成为所有孤儿进程的父进程
- 负责回收僵尸进程
- 系统关机时最后被终止
- 内核会确保PID 1进程不会意外退出
</details>

**2. 解释Android属性的命名规则和访问权限机制**
<details>
<summary>提示：关注属性前缀的含义</summary>

答案：
- `ro.*`：只读属性，启动时设置后不可修改
- `persist.*`：持久化到`/data/property`，重启保留
- `sys.*`：系统控制属性，通常触发系统行为
- `debug.*`：调试属性，生产版本可能被忽略
- 访问权限通过SELinux和UID/GID控制
- 属性设置需要通过property_service验证
</details>

**3. RC脚本中的`class`和`disabled`选项有什么作用？**
<details>
<summary>提示：考虑服务的批量管理和按需启动</summary>

答案：
- `class`：将服务分组，可通过`class_start`/`class_stop`批量控制
- `disabled`：服务不会自动启动，需要显式`start`命令或属性触发
- 结合使用可实现分阶段启动和条件启动
- 常见class：core、hal、late_start等
</details>

**4. First Stage Init为什么要以最小化方式实现？**
<details>
<summary>提示：考虑安全性和可靠性</summary>

答案：
- SELinux尚未加载，系统处于不安全状态
- 文件系统未完全就绪，功能受限
- 减少攻击面，避免安全漏洞
- 确保关键初始化步骤的可靠性
- 失败后系统无法恢复，必须极其稳定
</details>

### 挑战题

**5. 设计一个RC脚本实现以下需求：当检测到调试版本时自动启动性能分析服务，但在用户版本中禁用**
<details>
<summary>提示：使用属性触发器和条件判断</summary>

答案：
```
service perf_daemon /system/bin/perfmond
    class late_start
    disabled
    user system
    group system
    
on property:ro.debuggable=1
    start perf_daemon
    
on property:ro.build.type=user
    stop perf_daemon
```
</details>

**6. 分析以下场景：某个关键服务反复崩溃重启，Init进程会如何处理？如何优化？**
<details>
<summary>提示：考虑服务重启策略和退避机制</summary>

答案：
Init进程的处理：
- 默认立即重启崩溃的服务
- 使用`restart_period`限制重启频率
- 记录重启次数到属性`init.svc.<name>.restarts`
- 可能触发`on service-exited`动作

优化方案：
- 设置`restart_period`和`timeout_period`
- 使用`critical`标记触发系统重启
- 添加`onrestart`执行清理操作
- 实现指数退避算法延迟重启
- 添加看门狗监控服务健康状态
</details>

**7. 比较Android的property机制与Windows注册表、iOS的NSUserDefaults，分析各自优劣**
<details>
<summary>提示：从性能、安全性、使用便利性等角度分析</summary>

答案：
Android Property：
- 优点：内存映射高性能，触发器机制灵活，权限控制细粒度
- 缺点：值类型单一（字符串），大小限制（92字节），无事务支持

Windows注册表：
- 优点：层次结构清晰，支持多种数据类型，有事务支持
- 缺点：性能开销大，容易污染，权限管理复杂

iOS NSUserDefaults：
- 优点：类型安全，与应用生命周期集成，自动同步iCloud
- 缺点：仅限应用内使用，无系统级配置，大数据性能差

使用场景：
- Android适合系统配置和跨进程通信
- Windows适合复杂配置管理
- iOS适合应用偏好设置
</details>

**8. 设计一个启动时间优化方案，将系统启动时间从10秒优化到5秒**
<details>
<summary>提示：分析启动瓶颈，考虑并行化和延迟加载</summary>

答案：
分析阶段：
1. 使用bootchart识别耗时服务
2. 分析I/O等待和CPU空闲时间
3. 检查服务依赖关系

优化策略：
1. **并行化**：
   - 独立服务改为不同class并行启动
   - 硬件初始化并行化（如Wi-Fi和蓝牙）
   
2. **延迟加载**：
   - 非关键服务延迟到开机动画后
   - 按需启动很少使用的服务
   
3. **I/O优化**：
   - 预读取常用文件到内存
   - 合并小文件减少寻道
   - 使用squashfs等压缩文件系统
   
4. **CPU优化**：
   - 启动时锁定最高频率
   - 关键服务绑定大核
   - 优化编译参数减少初始化开销
   
5. **厂商定制**：
   - 实现suspend-to-disk快速恢复
   - 硬件级启动加速（bootloader优化）
   - AI预测用户常用服务优先加载
</details>

## 常见陷阱与错误 (Gotchas)

1. **RC脚本语法错误**
   - 错误：使用Tab缩进（必须用空格）
   - 错误：命令参数包含未转义的特殊字符
   - 错误：service名称包含点号或斜杠

2. **属性设置失败**
   - 原因：SELinux拒绝访问
   - 原因：属性名超过31字符限制
   - 原因：属性值超过91字符限制

3. **服务启动失败**
   - 忘记设置正确的文件权限和SELinux标签
   - 依赖的资源未就绪（如/data未挂载）
   - 用户/组不存在或权限不足

4. **SELinux相关**
   - 在permissive模式下测试正常，enforcing模式失败
   - 忘记更新file_contexts导致标签错误
   - 域转换失败导致服务运行在错误上下文

5. **性能陷阱**
   - 同步等待导致启动串行化
   - 过早启动大量服务导致资源竞争
   - 频繁的属性变化触发过多action

## 最佳实践检查清单

### RC脚本设计
- [ ] 使用有意义的service和action名称
- [ ] 合理设置service的class分组
- [ ] 必要时使用disabled+触发器启动
- [ ] 设置合适的oom_score_adjust保护关键服务
- [ ] 使用capabilities替代root权限

### 属性使用
- [ ] 遵循命名规范（如厂商属性使用vendor.前缀）
- [ ] 敏感信息不要存储在属性中
- [ ] 使用persist.*保存需要持久化的配置
- [ ] 避免高频属性更新（考虑性能影响）

### SELinux策略
- [ ] 最小权限原则，避免过度授权
- [ ] 使用neverallow规则防止策略违规
- [ ] 定期审计audit日志发现潜在问题
- [ ] 测试enforcing模式下的完整功能

### 启动优化
- [ ] 识别并优化关键路径上的服务
- [ ] 延迟非必要服务到系统空闲时启动
- [ ] 使用工具量化优化效果
- [ ] 在不同硬件配置上测试启动时间

### 调试技巧
- [ ] 使用`getprop`和`setprop`调试属性问题
- [ ] 查看`/proc/1/`了解init进程状态
- [ ] 使用`dmesg`查看内核日志中的SELinux拒绝
- [ ] 保存bootchart数据用于性能分析