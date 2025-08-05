# 第4章：Init进程与系统启动

Init进程是Android系统中第一个用户空间进程，承担着整个系统的初始化重任。本章将深入剖析Init进程的实现原理，包括其启动流程、RC脚本解析、属性服务、SELinux策略加载等核心机制。通过与Linux传统init、iOS的launchd、鸿蒙OS的init对比，读者将全面理解Android独特的系统启动架构。

## 4.1 Init进程源码分析

### 4.1.1 Init进程的诞生

当Linux内核完成自身初始化后，会启动第一个用户空间进程——Init进程（PID=1）。Android的Init进程实现位于`system/core/init/`目录，其入口函数是`main()`。内核通过`run_init_process()`调用`/init`二进制文件，这是ramdisk中的第一个程序。

**内核到用户空间的转换**
```
kernel_init() -> run_init_process("/init") -> execve("/init")
```

内核在`kernel_init()`函数中按以下顺序尝试启动init：
1. `ramdisk_execute_command`（通常是`/init`）
2. `/sbin/init`
3. `/etc/init`
4. `/bin/init`
5. `/bin/sh`（紧急模式）

Android将init放在ramdisk根目录，确保第一个被找到。内核传递的环境变量极其有限：
- `HOME=/`
- `TERM=linux`
- `PATH=/sbin:/usr/sbin:/bin:/usr/bin`

Init进程的生命周期分为两个阶段：

**First Stage Init**
First Stage Init在ramdisk环境中运行，使用静态链接的最小化二进制文件，主要任务是准备真正的根文件系统：

- 挂载基础文件系统（/dev、/proc、/sys等）
  - 使用`mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755")`
  - 创建必要的子目录：/dev/pts、/dev/socket、/dev/dm-user
  - 挂载`mount("devpts", "/dev/pts", "devpts", 0, NULL)`支持伪终端
  - 挂载`mount("proc", "/proc", "proc", 0, "hidepid=2")`增强安全性
  - 挂载`mount("sysfs", "/sys", "sysfs", 0, NULL)`导出内核对象
  
- 创建设备节点
  - `/dev/null`、`/dev/zero`、`/dev/full`等基础设备
  - `/dev/kmsg`用于内核日志（主设备号1，次设备号11）
  - `/dev/random`、`/dev/urandom`用于随机数（主设备号1，次设备号8/9）
  - `/dev/ptmx`伪终端主设备（主设备号5，次设备号2）
  - 使用`mknod()`系统调用创建，设置权限0666或0600
  
- 初始化内核日志
  - 通过`android::base::InitLogging()`设置日志系统
  - 重定向stdout/stderr到`/dev/kmsg`
  - 设置日志级别和格式（时间戳、进程信息等）
  - 配置klog过滤规则，避免日志风暴
  
- 挂载必要的块设备
  - 解析设备树（device tree）获取分区信息
  - 等待块设备就绪（通过uevent监听）
  - 挂载system分区到`/system`（只读）
  - 挂载vendor分区到`/vendor`（只读）
  - 处理A/B分区的slot选择
  
- 加载SELinux策略（如果是强制模式）
  - 调用`SelinuxInitialize()`加载编译后的策略
  - 设置进程上下文为`u:r:init:s0`
  - 从`/system/etc/selinux/plat_sepolicy.cil`加载平台策略
  - 合并`/vendor/etc/selinux/vendor_sepolicy.cil`厂商策略
  - 通过`selinux_android_load_policy()`加载到内核
  
- 执行Second Stage Init
  - 通过`execv("/system/bin/init", argv)`重新执行自身
  - 传递`--second-stage`参数标识
  - 保留必要的文件描述符（如`/dev/kmsg`）

**Second Stage Init**
Second Stage Init从真正的系统分区运行，拥有完整的库支持和更丰富的功能：

- 初始化属性服务（Property Service）
  - 创建属性共享内存区域（`/dev/__properties__`）
  - 初始化属性区域头部，设置魔数和版本号
  - 启动Unix域套接字监听属性设置请求（`/dev/socket/property_service`）
  - 加载默认属性文件：
    - `/default.prop`（ramdisk中的属性）
    - `/system/build.prop`（系统属性）
    - `/vendor/build.prop`（厂商属性）
    - `/product/build.prop`（产品属性）
    - `/odm/build.prop`（ODM属性）
  - 设置属性区域的SELinux标签和访问权限
  
- 解析RC脚本
  - 从以下目录按顺序加载RC文件：
    - `/init.rc`（主配置文件）
    - `/system/etc/init/`（系统服务）
    - `/vendor/etc/init/`（厂商服务）
    - `/odm/etc/init/`（ODM服务）
  - 构建Action和Service的内部数据结构
  - 验证语法正确性和权限要求
  - 检查服务的可执行文件是否存在
  - 预处理import语句，支持属性值替换
  
- 初始化子系统
  - 启动ueventd处理设备事件（软链接到init）
  - 启动watchdogd防止系统挂起（如果硬件支持）
  - 创建控制消息处理器（`/dev/socket/init`）
  - 注册信号处理器（SIGCHLD、SIGTERM等）
  
- 启动系统服务
  - 根据class和触发器有序启动
  - 设置进程的capabilities、优先级、cgroup等
  - 配置OOM调整值保护关键服务
  - 设置进程的namespace隔离（如果配置）
  
- 进入主事件循环
  - 基于epoll的事件驱动模型
  - 处理属性变化、子进程退出、控制消息等
  - 执行pending的Action队列
  - 监控服务健康状态，必要时重启

**关键环境变量设置**
```
PATH=/system/bin:/system/xbin:/vendor/bin:/odm/bin
LD_LIBRARY_PATH=/system/lib64:/vendor/lib64:/odm/lib64
ANDROID_ROOT=/system
ANDROID_DATA=/data
ANDROID_STORAGE=/storage
ANDROID_RUNTIME_ROOT=/apex/com.android.runtime
ANDROID_TZDATA_ROOT=/apex/com.android.tzdata
ANDROID_ART_ROOT=/apex/com.android.art
BOOTCLASSPATH=/apex/com.android.runtime/javalib/core-oj.jar:...
```

### 4.1.2 关键数据结构

Init进程使用精心设计的数据结构管理系统启动，这些结构体现了Android对启动过程的细粒度控制：

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
    
    // 触发器管理
    std::set<std::string> required_property_contexts;
    bool TriggersEqual(const Action& other) const;
    
    // 执行控制
    std::chrono::steady_clock::time_point last_command_time;
    std::optional<std::chrono::milliseconds> command_timeout;
}
```

Action的关键特性：
- **触发器类型**：事件触发器（如`boot`）或属性触发器（如`property:sys.boot_completed=1`）
- **命令队列**：支持异步执行，每次循环只执行一个命令，避免阻塞
- **优先级控制**：相同触发器的Action按解析顺序执行
- **一次性执行**：`oneshot`标记的Action只执行一次

**Service（服务）**
```cpp
struct Service {
    std::string name;
    unsigned flags;  // SVC_RUNNING, SVC_DISABLED, SVC_RESTARTING等
    pid_t pid;
    std::vector<std::string> args;
    std::vector<Option> options;
    
    // 服务控制
    int crash_count;
    time_t time_started;
    time_t time_crashed;
    std::chrono::seconds restart_period = 5s;  // 重启间隔
    std::optional<std::chrono::seconds> timeout_period;  // 启动超时
    
    // 进程管理
    std::optional<std::string> console;  // 控制台设备
    std::optional<bool> sigstop;  // 启动后是否暂停
    std::vector<std::pair<int, rlimit>> rlimits;  // 资源限制
    std::vector<std::string> writepid_files;  // PID文件路径
    
    // 安全上下文
    std::string seclabel;  // SELinux标签
    std::vector<uid_t> supp_gids;  // 补充组ID
    CapSet capabilities;  // Linux capabilities
    std::optional<std::vector<std::string>> updatable;  // APEX更新
    
    // 命名空间隔离
    unsigned namespace_flags;  // CLONE_NEWPID, CLONE_NEWNET等
    std::vector<std::string> namespaces_to_enter;  // 进入已存在的namespace
    
    // cgroup和调度
    std::map<std::string, std::string> cgroup_paths;
    int oom_score_adjust = -1000;  // OOM killer调整值
    int priority = 0;  // nice值
    struct sched_param scheduling_param;  // 实时调度参数
    
    // 服务依赖
    std::set<std::string> before_services;  // 必须在这些服务前启动
    std::set<std::string> after_services;   // 必须在这些服务后启动
    
    // 关键服务处理
    bool is_critical = false;  // 崩溃是否触发重启
    std::vector<std::string> critical_services_to_restart;
}
```

Service的高级特性：
- **生命周期管理**：自动重启、崩溃计数、退避算法
- **资源隔离**：支持Linux namespace、cgroup、capabilities
- **依赖关系**：服务间的启动顺序依赖
- **更新支持**：与APEX模块系统集成

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
    atomic_uint_least32_t serial;  // 全局版本号
    uint32_t magic;  // 0x504f5250 ('PROP')
    uint32_t version;  // 当前版本号
    uint32_t reserved[4];
    char data[0];  // 实际属性数据
};

// 属性节点（Trie树结构）
struct prop_bt {
    uint32_t namelen;  // 名称长度
    uint32_t proplen;  // 属性数量
    uint32_t children;  // 子节点偏移
    uint32_t props;  // 属性列表偏移
    char name[0];  // 节点名称
};

// 属性上下文映射
struct prop_context {
    const char* name_prefix;  // 属性前缀
    const char* context;  // SELinux上下文
    bool exact_match;  // 是否精确匹配
};
```

系统属性的高级特性：
- **共享内存实现**：通过`mmap`映射到所有进程，零拷贝访问
- **Trie树索引**：支持前缀查询和通配符匹配
- **版本控制**：通过`serial`字段检测属性变化，避免轮询
- **分区存储**：不同前缀的属性存储在独立的内存区域
- **访问控制**：结合SELinux和UID权限进行细粒度控制

**命令管理器（CommandManager）**
```cpp
class BuiltinFunctionMap {
    std::map<std::string, BuiltinFunction> functions_;
    
    // 注册的内置命令
    void RegisterBuiltinFunctions() {
        // 文件系统操作
        Register("chmod", do_chmod);
        Register("chown", do_chown);
        Register("copy", do_copy);
        Register("mkdir", do_mkdir);
        Register("mount", do_mount);
        Register("umount", do_umount);
        Register("symlink", do_symlink);
        Register("rm", do_rm);
        Register("rmdir", do_rmdir);
        Register("write", do_write);
        
        // 服务控制
        Register("class_start", do_class_start);
        Register("class_stop", do_class_stop);
        Register("class_reset", do_class_reset);
        Register("start", do_start);
        Register("stop", do_stop);
        Register("restart", do_restart);
        Register("enable", do_enable);
        
        // 属性操作
        Register("setprop", do_setprop);
        Register("getprop", do_getprop);
        Register("wait_for_prop", do_wait_for_prop);
        
        // 进程执行
        Register("exec", do_exec);
        Register("exec_start", do_exec_start);
        Register("exec_background", do_exec_background);
        
        // 触发器管理
        Register("trigger", do_trigger);
        
        // 系统控制
        Register("bootchart", do_bootchart);
        Register("init_user0", do_init_user0);
        Register("installkey", do_installkey);
        Register("load_persist_props", do_load_persist_props);
        
        // 网络配置
        Register("hostname", do_hostname);
        Register("domainname", do_domainname);
        Register("ifup", do_ifup);
        
        // 安全相关
        Register("restorecon", do_restorecon);
        Register("restorecon_recursive", do_restorecon_recursive);
        Register("setrlimit", do_setrlimit);
        Register("swapon_all", do_swapon_all);
        
        // 日志和调试
        Register("loglevel", do_loglevel);
        Register("mark_post_data", do_mark_post_data);
    }
    
    // 命令执行包装
    Result<Success> Execute(const std::string& name, 
                           const std::vector<std::string>& args) {
        auto it = functions_.find(name);
        if (it == functions_.end()) {
            return Error() << "Unknown command: " << name;
        }
        return it->second(args);
    }
};
```

命令系统的设计特点：
- **模块化设计**：每个命令是独立的函数，易于扩展
- **错误处理**：使用`Result<T>`类型安全地返回错误
- **参数验证**：每个命令函数负责验证自己的参数
- **异步支持**：`exec_background`等命令支持非阻塞执行

### 4.1.3 事件循环机制

Init进程的主循环采用高效的epoll机制，这是Android系统响应性的基础。与传统的select/poll相比，epoll在大量文件描述符场景下性能更优：

**事件源类型**
Init进程监听的事件源包括：
- 属性设置请求（通过Unix域套接字`/dev/socket/property_service`）
- 子进程退出信号（通过signalfd将SIGCHLD转换为文件描述符事件）
- 控制命令（通过`/dev/socket/init`接收如`ctl.start`等命令）
- 内核消息（通过netlink套接字接收uevent等）
- 看门狗定时器（防止init进程挂起）
- keychord事件（组合键触发特殊动作）

**Epoll架构实现**
```cpp
class Epoll {
private:
    int epoll_fd_;
    std::map<int, std::function<void()>> handlers_;
    
public:
    void Open() {
        // 创建epoll实例，EPOLL_CLOEXEC防止fd泄露到子进程
        epoll_fd_ = epoll_create1(EPOLL_CLOEXEC);
        if (epoll_fd_ == -1) {
            PLOG(FATAL) << "epoll_create1 failed";
        }
        
        // 注册信号处理（使用signalfd）
        sigset_t mask;
        sigemptyset(&mask);
        sigaddset(&mask, SIGCHLD);
        sigaddset(&mask, SIGTERM);
        signal_fd_ = signalfd(-1, &mask, SFD_CLOEXEC);
        RegisterHandler(signal_fd_, HandleSignals);
        
        // 注册属性服务
        property_fd_ = CreatePropertySocket();
        RegisterHandler(property_fd_, HandlePropertySet);
        
        // 注册控制消息
        init_fd_ = CreateInitSocket();
        RegisterHandler(init_fd_, HandleInitSocket);
        
        // 注册keychord（组合键）
        keychord_fd_ = OpenKeychordDevice();
        if (keychord_fd_ >= 0) {
            RegisterHandler(keychord_fd_, HandleKeychord);
        }
    }
    
    void RegisterHandler(int fd, std::function<void()> handler) {
        epoll_event ev = {};
        ev.events = EPOLLIN;  // 监听可读事件
        ev.data.fd = fd;
        
        if (epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, fd, &ev) == -1) {
            PLOG(ERROR) << "Failed to add fd " << fd << " to epoll";
            return;
        }
        
        handlers_[fd] = std::move(handler);
    }
    
    std::optional<std::function<void()>> Wait(
            std::optional<std::chrono::milliseconds> timeout) {
        epoll_event events[32];
        int timeout_ms = -1;
        
        if (timeout.has_value()) {
            timeout_ms = timeout->count();
        }
        
        int nr = epoll_wait(epoll_fd_, events, 
                           arraysize(events), timeout_ms);
        
        if (nr == -1) {
            PLOG(ERROR) << "epoll_wait failed";
            return std::nullopt;
        }
        
        for (int i = 0; i < nr; i++) {
            auto it = handlers_.find(events[i].data.fd);
            if (it != handlers_.end()) {
                return it->second;
            }
        }
        
        return std::nullopt;
    }
};
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