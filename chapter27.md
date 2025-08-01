# 第27章：实时性与性能优化

Android系统的流畅性直接影响用户体验，本章深入剖析Android的实时性保证机制和性能优化策略。我们将从Linux内核调度器的Android定制开始，探讨如何通过RT调度、Jank检测、内存管理和功耗优化等技术手段，打造一个响应迅速、运行流畅的移动操作系统。通过与iOS、鸿蒙等竞争系统的对比，理解Android在实时性和性能优化方面的独特设计。

## 1. RT调度器应用

### 1.1 Android调度器架构概述

Android基于Linux内核的调度器，但针对移动设备的特点进行了大量优化。理解这些优化对于开发高性能应用至关重要。

#### CFS (Completely Fair Scheduler) 基础

Android默认使用CFS调度器，它通过红黑树维护可运行进程队列，使用虚拟运行时间(vruntime)保证公平性。关键概念包括：

- **nice值与权重**：nice值范围-20到19，通过`prio_to_weight[]`数组转换为权重
- **时间片计算**：基于`sched_latency_ns`和进程数量动态计算
- **虚拟运行时间**：`vruntime = 实际运行时间 * NICE_0_LOAD / 进程权重`
- **唤醒抢占**：通过`sysctl_sched_wakeup_granularity`控制抢占粒度

Android对CFS的主要修改：
- 调整`sched_latency_ns`从6ms到10ms，适应移动设备特性
- 修改`sched_min_granularity_ns`提高交互响应
- 引入`schedtune`机制进行能效调优

#### RT调度类实现

RT调度类为实时任务提供确定性调度保证，Android中主要用于：
- 音频处理线程(AudioFlinger)
- 触摸事件处理(InputDispatcher)
- 显示合成(SurfaceFlinger的关键路径)
- 相机预览线程

RT调度特点：
- **优先级范围**：1-99，数值越大优先级越高
- **调度策略**：SCHED_FIFO和SCHED_RR
- **抢占规则**：高优先级任务总是抢占低优先级
- **CPU带宽限制**：通过`sched_rt_runtime_us`和`sched_rt_period_us`防止RT任务独占CPU

#### EAS (Energy Aware Scheduling) 集成

Android 5.0开始集成EAS，实现性能与功耗的平衡：

- **能效模型**：通过设备树定义CPU能效曲线
- **任务放置**：`select_energy_cpu_idx()`选择最优CPU
- **负载追踪**：PELT (Per-Entity Load Tracking)算法
- **thermal压力**：温度限制下的性能调整

关键函数：
- `find_energy_efficient_cpu()`: EAS核心决策函数
- `compute_energy()`: 计算任务在特定CPU上的能耗
- `schedutil_cpu_freq()`: 频率调节接口

#### 与Linux主线调度器差异

Android调度器的主要差异：
1. **schedtune控制器**：提供per-cgroup的性能提升
2. **prefer_idle**：优先选择空闲CPU
3. **uclamp**：细粒度的utilization clamping
4. **RTG (Related Thread Groups)**：相关线程组优化

### 1.2 实时线程管理

#### RT优先级设计

Android系统服务的RT优先级分配遵循严格规范：

```
优先级分配原则：
99: 仅用于关键中断处理
95-98: 音频HAL回调线程
90-94: 触摸事件处理
85-89: 显示相关线程
80-84: 相机服务
1-79: 其他RT需求
```

关键API：
- `setpriority()`: 设置nice值
- `sched_setscheduler()`: 设置调度策略和RT优先级
- `pthread_setschedparam()`: POSIX线程接口
- `android_os_Process_setThreadScheduler()`: Android特有接口

#### 音频线程实时性保证

音频是Android中对实时性要求最高的子系统：

**FastMixer线程**：
- RT优先级：通常为95-98
- CPU亲和性：绑定到大核
- 缓冲区：仅2-3个period，约5-10ms
- 中断处理：ALSA驱动直接唤醒

**音频策略**：
- `AudioFlinger::MixerThread`使用SCHED_FIFO
- HAL回调线程享有最高RT优先级
- 通过`audio.offload.min.duration.secs`控制offload阈值

优化技巧：
- 使用`ANDROID_PRIORITY_AUDIO`宏
- 避免在音频线程中分配内存
- 使用lock-free数据结构
- 监控`audio.flinger.log.latency`

#### 渲染线程优先级调优

UI渲染流畅性直接影响用户体验：

**RenderThread配置**：
- 默认nice值：-10到-4
- 动画期间动态提升
- 与主线程协同调度

**HWUI渲染管线**：
1. UI线程：构建DisplayList
2. RenderThread：OpenGL命令生成
3. GPU：异步渲染
4. SurfaceFlinger：合成显示

优化要点：
- `debug.hwui.render_thread_priority`调整优先级
- 监控`gfx.view.stats`了解渲染性能
- 使用`dumpsys gfxinfo`分析帧时间

#### 触摸响应优化

触摸延迟是用户最敏感的性能指标：

**InputDispatcher优化**：
- RT优先级90+
- CPU boost触发
- 预测性触摸处理

**延迟优化技术**：
1. **硬件层**：高采样率触摸屏(120Hz+)
2. **驱动层**：中断亲和性绑定
3. **框架层**：批处理与预测
4. **应用层**：异步触摸处理

关键指标：
- Motion-to-photon延迟目标：<20ms
- 触摸采样率：120-240Hz
- 预测算法：Kalman滤波器

### 1.3 CPU亲和性与大小核调度

#### big.LITTLE架构支持

Android深度优化了ARM big.LITTLE架构支持：

**集群类型**：
- **传统big.LITTLE**：2+4或4+4配置
- **DynamIQ**：灵活的1+3+4等配置
- **三集群**：超大核+大核+小核

**调度域构建**：
```
MC domain: 同集群内CPU
DIE domain: 跨集群调度
NUMA domain: 多芯片系统
```

**能效感知**：
- `capacity`：CPU算力标定
- `capacity_orig`：最大算力
- `capacity_curr`：当前算力(考虑频率和温度)

#### 任务迁移策略

任务在大小核间迁移的决策机制：

**迁移触发条件**：
1. **负载不均衡**：`load_balance()`周期性检查
2. **唤醒时**：`select_task_rq()`选择目标CPU
3. **能效优化**：EAS主动迁移
4. **温度限制**：thermal governor介入

**迁移成本考虑**：
- Cache亲和性损失
- 跨集群通信开销
- 功耗状态切换延迟

**HMP (Heterogeneous Multi-Processing) 策略**：
- `sched_upmigrate`：任务迁移到大核阈值
- `sched_downmigrate`：任务迁移到小核阈值
- `sched_small_task`：小任务判定标准

#### 热插拔机制

CPU热插拔用于极限省电场景：

**核心组件**：
- `cpu_subsys`：sysfs接口
- `cpuhp_state_machine`：状态机管理
- `cpufreq_governor`：与调频协同

**Android定制**：
1. **msm_performance**：高通平台优化
2. **core_ctl**：智能核心控制
3. **isolation**：CPU隔离机制

**优化建议**：
- 避免频繁热插拔
- 考虑唤醒延迟
- 监控`trace_cpu_hotplug`

#### DynamIQ集群管理

ARM DynamIQ带来更灵活的CPU配置：

**关键特性**：
- 单集群内异构CPU
- 共享L3 cache
- 精细功耗控制

**调度适配**：
- `phantom domains`：虚拟调度域
- `capacity_margin`：预留算力
- `misfit task`：任务不匹配检测

**性能优化**：
- Cache分区(CAP)
- 内存延迟优化
- 中断亲和性调优

## 2. Jank检测与优化

### 2.1 Jank产生机制分析

Jank（卡顿）是指UI渲染无法跟上显示刷新率，导致的视觉不连续现象。理解Jank产生的根本原因是优化的第一步。

#### 帧渲染管线

Android的图形渲染采用生产者-消费者模型，涉及多个阶段：

**渲染管线阶段**：
1. **Input**：触摸事件产生和分发
2. **Animation**：动画计算和属性更新
3. **Measure/Layout**：视图树测量和布局
4. **Draw**：构建DisplayList
5. **Sync**：同步RenderThread
6. **Render**：GPU命令生成
7. **Swap**：缓冲区交换
8. **Composite**：SurfaceFlinger合成

**关键时间节点**：
- `Vsync`：垂直同步信号，60Hz屏幕每16.67ms一次
- `Input Latency`：触摸到响应延迟
- `Frame Deadline`：帧必须完成的最后期限
- `Present Time`：帧实际显示时间

**性能监控点**：
- `getFrameStats()`: 获取帧统计信息
- `FrameMetrics API`: 详细的帧时间分解
- `dumpsys gfxinfo`: 系统级图形信息

#### VSYNC机制

VSYNC是Android图形系统的心跳，协调整个渲染流程：

**VSYNC分发机制**：
```
HW Composer → SurfaceFlinger → Choreographer → App
```

**三种VSYNC**：
1. **HW VSYNC**：硬件产生的真实信号
2. **SW VSYNC**：软件模拟的VSYNC
3. **App VSYNC**：应用接收的VSYNC，有phase offset

**Phase offset优化**：
- `VSYNC_EVENT_PHASE_OFFSET_NS`：SurfaceFlinger偏移
- `SF_VSYNC_EVENT_PHASE_OFFSET_NS`：应用偏移
- 目的：给渲染更多时间，减少延迟

**DispSync模型**：
- 预测下一个VSYNC时间
- 处理VSYNC漂移
- 支持可变刷新率(VRR)

#### Triple Buffering

Android使用多缓冲技术平衡性能和延迟：

**缓冲区角色**：
1. **Front Buffer**：正在显示的缓冲区
2. **Back Buffer**：正在渲染的缓冲区
3. **Third Buffer**：备用缓冲区，处理掉帧

**BufferQueue机制**：
- `dequeueBuffer()`: 获取可用缓冲区
- `queueBuffer()`: 提交渲染完成的缓冲区
- `acquireBuffer()`: SurfaceFlinger获取待合成缓冲区

**优化策略**：
- 动态调整缓冲区数量
- 根据渲染压力切换双缓冲/三缓冲
- 监控`buffer starvation`

#### 掉帧原因分类

深入理解各种掉帧原因有助于针对性优化：

**1. CPU限制**：
- 主线程阻塞（网络、磁盘IO）
- 布局层级过深
- 过度绘制
- 频繁GC

**2. GPU限制**：
- 复杂shader
- 大纹理上传
- 过多绘制调用
- GPU内存带宽饱和

**3. 系统资源竞争**：
- 后台任务抢占CPU
- 内存压力触发回收
- 温度限制降频
- 其他应用干扰

**4. 框架问题**：
- Choreographer调度延迟
- SurfaceFlinger合成瓶颈
- Binder通信阻塞
- 锁竞争

### 2.2 Systrace与Perfetto工具链

性能分析工具是Jank优化的利器，Android提供了强大的工具链。

#### Atrace架构

Atrace是Android的系统级跟踪框架：

**核心组件**：
- `kernel/trace/trace.c`：内核跟踪基础设施
- `libcutils/trace.c`：用户空间跟踪API
- `atrace`命令：数据收集工具

**跟踪类别**：
```
gfx: 图形系统
input: 输入系统
view: View系统
wm: 窗口管理器
am: 活动管理器
sm: 同步管理器
audio: 音频系统
video: 视频系统
camera: 相机系统
```

**使用方式**：
- `ATRACE_CALL()`: 函数级跟踪
- `ATRACE_BEGIN/END()`: 代码块跟踪
- `ATRACE_ASYNC_BEGIN/END()`: 异步事件跟踪
- `ATRACE_INT()`: 计数器跟踪

#### Perfetto数据收集

Perfetto是新一代跟踪系统，提供更强大的功能：

**架构优势**：
- 统一的数据模型(protobuf)
- 低开销的进程内跟踪
- 灵活的触发机制
- 长时间跟踪支持

**核心组件**：
1. **traced**：系统守护进程
2. **traced_probes**：数据源服务
3. **perfetto SDK**：应用集成库
4. **trace processor**：数据分析引擎

**数据源类型**：
- `linux.ftrace`: 内核事件
- `android.packages_list`: 包信息
- `android.process_stats`: 进程统计
- `android.gpu.memory`: GPU内存
- `track_event`: 自定义事件

**配置示例**：
```
TraceConfig {
  duration_ms: 10000
  buffers {
    size_kb: 65536
  }
  data_sources {
    config {
      name: "linux.ftrace"
      ftrace_config {
        ftrace_events: "sched/*"
        ftrace_events: "power/*"
      }
    }
  }
}
```

#### 性能指标分析

关键性能指标帮助量化Jank程度：

**帧时间指标**：
- **Frame Duration**：总帧时间
- **Janky Frames**：超过阈值的帧
- **99th Percentile**：最差1%的帧时间
- **Frame Uniformity**：帧时间方差

**计算公式**：
```
Jank率 = Janky帧数 / 总帧数
帧时间预算 = 1000ms / 刷新率
超时帧 = 实际帧时间 > 帧时间预算
```

**分析维度**：
1. **时间分解**：各阶段耗时
2. **线程分析**：CPU利用率和等待
3. **系统负载**：CPU/GPU/内存使用率
4. **热点识别**：最耗时的函数

#### 自动化Jank检测

自动化检测帮助持续监控性能：

**JankStats库**：
```java
JankStats.createAndTrack(
    window,
    frameListener = { frameData ->
        if (frameData.isJank) {
            // 记录Jank信息
        }
    }
)
```

**FrameMetricsAggregator**：
- 收集详细帧指标
- 支持直方图分析
- 可导出到APM系统

**CI/CD集成**：
- Monkey测试 + Systrace
- 自动化性能回归检测
- 基准测试(Macrobenchmark)

### 2.3 Choreographer优化

Choreographer是Android动画和UI更新的中枢，优化它对改善Jank至关重要。

#### 帧调度机制

Choreographer负责协调所有UI更新：

**回调类型**：
1. **INPUT**：输入事件处理
2. **ANIMATION**：动画更新
3. **INSETS_ANIMATION**：窗口动画
4. **TRAVERSAL**：布局和绘制
5. **COMMIT**：提交帧

**调度流程**：
```java
1. scheduleVsyncLocked() → 请求VSYNC
2. onVsync() → 接收VSYNC信号  
3. doFrame() → 执行帧回调
4. doCallbacks() → 按类型执行回调
```

**优化要点**：
- 避免在UI线程执行耗时操作
- 使用`postFrameCallback()`而非`post()`
- 监控`skippedFrames`识别卡顿

#### Input latency优化

减少输入延迟提升交互体验：

**延迟组成**：
1. **硬件延迟**：触摸屏扫描
2. **驱动延迟**：中断处理
3. **系统延迟**：事件分发
4. **应用延迟**：事件处理
5. **渲染延迟**：UI更新

**优化技术**：
- **输入预测**：基于历史数据预测
- **批处理优化**：合并相近事件
- **优先级提升**：输入线程RT调度
- **早期唤醒**：提前唤醒渲染线程

**测量方法**：
```java
window.addOnFrameMetricsAvailableListener(
    { window, frameMetrics, dropCount ->
        val inputDelay = frameMetrics.getMetric(
            FrameMetrics.INPUT_HANDLING_DURATION
        )
    }
)
```

#### 动画性能调优

流畅动画是良好用户体验的关键：

**动画类型优化**：
1. **属性动画**：使用`RenderNodeAnimator`
2. **过渡动画**：缓存场景状态
3. **物理动画**：使用`SpringAnimation`
4. **Lottie动画**：硬件加速渲染

**优化技巧**：
- 使用`ValueAnimator.setFrameDelay()`调整帧率
- 避免在动画回调中创建对象
- 使用`ViewPropertyAnimator`批量更新
- 开启硬件加速`setLayerType(LAYER_TYPE_HARDWARE)`

**动画监控**：
```java
animator.addUpdateListener { animation ->
    // 避免复杂计算
    view.translationX = animation.animatedValue as Float
}
```

#### RenderThread优化

RenderThread负责GPU命令生成：

**线程特性**：
- 独立于UI线程
- 拥有独立的GL context
- 支持异步渲染

**优化策略**：
1. **减少绘制复杂度**：
   - 简化自定义View
   - 使用`Canvas.quickReject()`
   - 避免`saveLayer()`

2. **纹理优化**：
   - 使用纹理图集
   - 及时释放纹理
   - 控制纹理大小

3. **命令优化**：
   - 批量绘制调用
   - 状态变化最小化
   - 使用Display List

**调试工具**：
- `debug.hwui.profile`: 显示渲染信息
- `debug.hwui.overdraw`: 过度绘制检测
- `dumpsys gfxinfo`: 详细渲染统计

## 3. 内存压力处理

### 3.1 低内存管理器演进

Android的低内存管理经历了从内核空间到用户空间的重要演进，这种变化带来了更灵活和智能的内存管理策略。

#### LMK到LMKD迁移

**LMK (Low Memory Killer) 时代**：

传统的LMK是内核模块，存在以下问题：
- 决策逻辑固定在内核中
- 难以适配不同设备
- 缺乏灵活的策略调整
- 无法利用用户空间信息

**LMKD (Low Memory Killer Daemon) 优势**：

LMKD作为用户空间守护进程，提供了：
- 更灵活的杀进程策略
- 可配置的内存阈值
- 与ActivityManager深度集成
- 支持多种压力信号源

**架构对比**：
```
LMK: Kernel → 直接杀进程
LMKD: Kernel → PSI/vmpressure → LMKD → ActivityManager → 杀进程
```

**LMKD工作流程**：
1. 监听内存压力事件
2. 计算当前内存状态
3. 选择牺牲进程
4. 通过socket通知kill
5. 更新系统状态

#### PSI (Pressure Stall Information) 集成

PSI是Linux 4.20引入的压力监控机制，Android充分利用了这一特性：

**PSI指标类型**：
- **some**: 至少一个任务阻塞
- **full**: 所有任务阻塞
- **avg10/60/300**: 不同时间窗口平均值

**内存压力监控**：
```
/proc/pressure/memory
some avg10=0.00 avg60=0.00 avg300=0.00 total=0
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

**LMKD中的PSI使用**：
- `init_psi_monitors()`: 初始化PSI监控
- `mp_event_psi()`: 处理PSI事件
- 阈值配置：`ro.lmk.psi_partial_stall_ms`
- 采样窗口：`ro.lmk.psi_complete_stall_ms`

**与vmpressure对比**：
- PSI更准确反映实际压力
- 支持更细粒度的监控
- 降低误杀率
- 更好的多核扩展性

#### 内存水位线调整

Android定义了多级内存水位线来触发不同的回收行为：

**水位线级别**：
```
Critical (致命) < Low (低) < Pressure (压力) < Medium (中等) < High (高)
```

**计算方式**：
- 基于`MemTotal`和`SwapTotal`
- 考虑zram压缩比
- 动态调整阈值

**关键属性**：
- `ro.lmk.critical`: 致命阈值(MB)
- `ro.lmk.low`: 低内存阈值
- `ro.lmk.medium`: 中等阈值
- `ro.lmk.critical_upgrade`: 是否允许升级

**自适应调整**：
```c
minfree = property_get_int32("ro.lmk.low", 1024);
if (mem_total < 2048) {
    minfree = minfree * mem_total / 2048;
}
```

#### 进程优先级oom_adj

oom_adj是决定进程生死的关键指标：

**adj值范围与含义**：
```
-1000: 不可杀死(系统关键进程)
-900: 持久进程(persistent)
-800: 系统进程
-700: 持久服务
0: 前台应用
100: 可见应用
200: 可感知应用
300: 备份应用
400-500: 服务进程
600-700: Home应用
800-900: 缓存进程
900+: 空进程
```

**计算因素**：
1. **进程状态**：前台/后台/服务
2. **组件状态**：Activity/Service/Provider
3. **客户端重要性**：绑定服务的客户端
4. **用户交互**：最近使用时间

**优化策略**：
- 合理使用前台服务
- 及时释放不需要的组件
- 避免内存泄漏提高存活率
- 使用JobScheduler延迟任务

### 3.2 内存回收策略

内存回收是维持系统流畅的关键机制，Android在Linux基础上进行了大量优化。

#### kswapd工作机制

kswapd是内核的内存回收线程，负责异步页面回收：

**触发条件**：
- 空闲页面低于`low`水位线
- 分配失败唤醒
- 定期唤醒检查

**回收流程**：
1. `balance_pgdat()`: 主循环
2. `shrink_zone()`: 回收特定zone
3. `shrink_lruvec()`: LRU链表回收
4. `shrink_slab()`: slab缓存回收

**Android优化**：
- 提高kswapd优先级
- 调整`swappiness`参数
- 优化`min_free_kbytes`
- 绑定CPU亲和性

**监控指标**：
```bash
cat /proc/vmstat | grep -E "pgsteal|pgscan|pgfree"
```

#### Direct reclaim优化

Direct reclaim是同步回收路径，直接影响应用性能：

**问题分析**：
- 阻塞内存分配
- 增加延迟
- 可能导致ANR

**优化措施**：
1. **减少触发**：
   - 预留更多内存
   - 提前异步回收
   - 使用内存池

2. **缩短时间**：
   - 限制回收页面数
   - 优先回收易释放页面
   - 避免回收代码页

3. **进程隔离**：
   - 关键进程使用`mlockall()`
   - 内存cgroup隔离
   - 专用内存预留

**内核参数调优**：
```bash
echo 0 > /proc/sys/vm/direct_reclaim_anon
echo 200 > /proc/sys/vm/direct_reclaim_swappiness
```

#### ZRAM压缩

ZRAM提供内存压缩功能，有效扩展可用内存：

**工作原理**：
- 创建基于RAM的块设备
- 压缩换出页面
- 透明解压访问

**配置优化**：
```bash
# 设置ZRAM大小
echo 2G > /sys/block/zram0/disksize
# 选择压缩算法
echo lz4 > /sys/block/zram0/comp_algorithm
# 启用ZRAM
mkswap /dev/block/zram0
swapon /dev/block/zram0
```

**压缩算法对比**：
- **lz4**: 最快，压缩比适中
- **lzo**: 平衡选择
- **zstd**: 压缩比最高，CPU开销大

**性能监控**：
```bash
cat /sys/block/zram0/mm_stat
# 输出：原始大小 压缩大小 内存使用 ...
```

#### ION内存池管理

ION是Android的统一内存分配器，管理各种内存池：

**内存池类型**：
1. **SYSTEM**: 普通内存
2. **SYSTEM_CONTIG**: 连续内存
3. **CARVEOUT**: 预留内存
4. **CMA**: 连续内存分配器
5. **DMA**: DMA缓冲区

**分配策略**：
```c
ion_alloc(len, heap_mask, flags)
→ 选择合适heap
→ 分配内存
→ 创建ion_buffer
→ 返回fd
```

**内存共享机制**：
- 基于fd的共享
- 支持跨进程映射
- 引用计数管理
- Cache一致性维护

**优化建议**：
- 合理选择heap类型
- 及时释放不用的buffer
- 使用cache维护API
- 监控碎片化情况

### 3.3 应用内存优化

应用层的内存优化对系统整体性能至关重要。

#### 内存泄漏检测

内存泄漏是导致应用性能下降和被杀的主要原因：

**常见泄漏类型**：
1. **Context泄漏**：静态引用Activity
2. **Handler泄漏**：内部类持有外部引用
3. **监听器泄漏**：未注销的监听器
4. **集合泄漏**：静态集合持有对象
5. **线程泄漏**：未结束的线程

**检测工具**：
- **LeakCanary**: 自动检测泄漏
- **Android Studio Profiler**: 实时内存分析
- **MAT (Memory Analyzer Tool)**: 深度分析
- **systrace**: 系统级内存追踪

**LeakCanary原理**：
1. 监控对象生命周期
2. 主动触发GC
3. 检查引用链
4. 生成泄漏报告

**防止泄漏的最佳实践**：
```java
// 使用弱引用
private static class MyHandler extends Handler {
    private final WeakReference<Activity> mActivity;
}

// 及时清理
@Override
protected void onDestroy() {
    handler.removeCallbacksAndMessages(null);
    unregisterReceiver(receiver);
}
```

#### Bitmap内存管理

Bitmap是Android应用最大的内存消耗源：

**内存计算**：
```
内存大小 = 宽度 × 高度 × 每像素字节数
ARGB_8888: 4字节/像素
RGB_565: 2字节/像素
```

**优化策略**：
1. **合适的解码配置**：
   ```java
   options.inSampleSize = 2; // 缩放
   options.inPreferredConfig = Bitmap.Config.RGB_565;
   ```

2. **复用Bitmap**：
   ```java
   options.inBitmap = reusableBitmap;
   options.inMutable = true;
   ```

3. **及时回收**：
   ```java
   bitmap.recycle();
   bitmap = null;
   ```

**高级优化**：
- 使用`BitmapRegionDecoder`加载大图
- 实现图片缓存(LRU)
- 使用WebP格式
- Hardware Bitmap(Android O+)

#### Native内存追踪

Native内存不受Java堆限制，需要特别关注：

**追踪工具**：
- **malloc_debug**: 内存分配追踪
- **memtrack**: HAL层内存统计  
- **perfetto**: 全面的内存追踪
- **AddressSanitizer**: 内存错误检测

**malloc_debug使用**：
```bash
adb shell setprop libc.debug.malloc.program app_process
adb shell setprop libc.debug.malloc.options backtrace=16
```

**常见问题**：
1. **内存泄漏**：未释放的malloc
2. **越界访问**：数组越界
3. **重复释放**：double free
4. **野指针**：use after free

**JNI内存管理**：
```c
// 创建全局引用
jobject globalRef = (*env)->NewGlobalRef(env, localRef);
// 及时删除
(*env)->DeleteGlobalRef(env, globalRef);
```

#### ART堆调优

ART运行时提供了多种堆调优参数：

**堆大小设置**：
```xml
<application android:largeHeap="true">
```

**运行时参数**：
- `-Xms`: 初始堆大小
- `-Xmx`: 最大堆大小  
- `-XX:HeapGrowthLimit`: 堆增长限制
- `-XX:HeapTargetUtilization`: 目标利用率

**GC调优**：
```java
// 主动触发GC
System.gc();
Runtime.getRuntime().gc();

// 获取内存信息
ActivityManager.MemoryInfo memInfo = new ActivityManager.MemoryInfo();
activityManager.getMemoryInfo(memInfo);
```

**分代管理**：
- **Young Generation**: 新对象分配
- **Old Generation**: 长期存活对象
- **Large Object Space**: 大对象专用

**监控指标**：
```bash
adb shell dumpsys meminfo <package>
# 查看详细内存使用情况
```

## 4. 功耗优化策略

### 4.1 CPU功耗管理

CPU是移动设备的主要功耗来源，Android通过多层次的功耗管理机制实现能效优化。

#### CPUFreq governor

CPUFreq子系统负责动态调整CPU频率，不同的governor实现不同的调频策略：

**常用Governor**：

1. **schedutil** (推荐)：
   - 与调度器深度集成
   - 基于CPU利用率调频
   - 响应更快，更省电
   
2. **interactive**：
   - Android传统选择
   - 快速响应用户交互
   - 可配置的升降频参数

3. **ondemand**：
   - Linux默认governor
   - 基于负载的调频
   - 相对保守

4. **performance/powersave**：
   - 固定最高/最低频率
   - 用于测试和特殊场景

**schedutil工作原理**：
```c
freq = max_freq * util / max_capacity
// util: CPU利用率
// max_capacity: CPU最大能力
```

**调优参数**：
- `up_rate_limit_us`: 升频限制时间
- `down_rate_limit_us`: 降频限制时间  
- `rate_limit_us`: 统一限制时间
- `hispeed_freq`: 高速频率阈值

**监控和调试**：
```bash
# 查看当前governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# 查看可用频率
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies

# 实时频率监控
watch -n 1 'cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq'
```

#### CPU Idle states

CPU空闲时进入不同深度的休眠状态以节省功耗：

**Idle状态级别**：
```
C0: Active (运行状态)
C1: WFI (Wait for Interrupt) - 最浅休眠
C2: 时钟关闭
C3: 电源门控
C4+: 深度休眠，可能关闭缓存
```

**cpuidle框架**：
- **Governor**: 决定进入哪个idle状态
- **Driver**: 实现具体的idle状态
- **QoS**: 延迟约束管理

**Android优化**：
1. **idle状态选择**：
   - 考虑唤醒延迟
   - 预测空闲时长
   - 平衡功耗和性能

2. **cluster idle**：
   - 整个CPU集群休眠
   - 更深的省电状态
   - 需要所有核心空闲

**调优建议**：
```bash
# 查看idle状态
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name

# 禁用深度idle(调试用)
echo 1 > /sys/devices/system/cpu/cpu0/cpuidle/state3/disable
```

#### 调频策略优化

根据不同场景优化调频策略：

**场景识别**：
1. **用户交互**：快速升频
2. **后台任务**：保守调频
3. **游戏模式**：性能优先
4. **省电模式**：限制最高频率

**Boost机制**：
```c
// Input boost
on_touch_event() {
    cpu_boost(300ms, 1.4GHz);
}

// Launch boost  
on_app_launch() {
    cpu_boost(1000ms, max_freq);
}
```

**频率表优化**：
- 移除低效频点
- 优化电压-频率曲线
- 考虑能效比

**与调度器协同**：
- EAS感知的调频
- 任务placement影响
- 多核协同调频

#### Turbo boost控制

现代CPU支持短时超频以提升突发性能：

**Turbo机制**：
- 温度允许时提升频率
- 功耗预算内最大化性能
- 单核/多核turbo区别

**Android管理**：
1. **thermal限制**：
   ```bash
   # thermal配置
   /vendor/etc/thermal-engine.conf
   ```

2. **功耗预算**：
   - 瞬时功耗限制
   - 平均功耗限制
   - 电池温度考虑

3. **应用场景**：
   - 应用启动加速
   - 相机拍照处理
   - 游戏加载场景

**监控指标**：
```bash
# CPU温度
cat /sys/class/thermal/thermal_zone*/temp

# 频率限制
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_max_freq
```

### 4.2 系统级功耗优化

Android提供了多种系统级的功耗优化机制。

#### Doze模式实现

Doze是Android 6.0引入的深度休眠机制：

**Doze状态机**：
```
ACTIVE → INACTIVE → IDLE_PENDING → SENSING → LOCATING → IDLE → IDLE_MAINTENANCE
```

**进入条件**：
1. 屏幕关闭
2. 电池供电
3. 静止不动
4. 未充电

**Doze限制**：
- 网络访问暂停
- WakeLock ignored
- Alarm延迟(除AlarmManager.setAlarmClock)
- WiFi扫描停止
- 同步适配器暂停
- JobScheduler延迟

**Light Doze**：
- 更宽松的限制
- 更快进入
- 保持网络
- 适合口袋场景

**白名单机制**：
```xml
<!-- 系统白名单 -->
/system/etc/sysconfig/

<!-- 应用请求 -->
<uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS"/>
```

**调试方法**：
```bash
# 强制进入Doze
adb shell dumpsys deviceidle force-idle

# 退出Doze
adb shell dumpsys deviceidle unforce

# 查看状态
adb shell dumpsys deviceidle
```

#### App Standby机制

App Standby限制不常用应用的后台活动：

**Standby分组** (Android P+)：
1. **Active**: 当前使用
2. **Working Set**: 经常使用
3. **Frequent**: 定期使用
4. **Rare**: 很少使用
5. **Restricted**: 限制最严

**分组依据**：
- 用户交互
- 通知interaction
- 前台服务
- 同步适配器活动

**限制内容**：
- 网络访问频率
- 作业运行频率
- 报警触发频率
- FCM优先级

**豁免条件**：
- 前台服务运行
- 设备充电
- 白名单应用

**API适配**：
```java
// 查询standby bucket
UsageStatsManager usm = getSystemService(UsageStatsManager.class);
int bucket = usm.getAppStandbyBucket();

// 监听变化
registerReceiver(new BroadcastReceiver() {
    public void onReceive(Context context, Intent intent) {
        // Standby状态改变
    }
}, new IntentFilter(UsageStatsManager.ACTION_STANDBY_BUCKET_CHANGED));
```

#### JobScheduler优化

JobScheduler提供了智能的后台任务调度：

**优化策略**：
1. **批处理执行**：
   - 合并相似任务
   - 统一唤醒时机
   - 减少唤醒次数

2. **条件约束**：
   ```java
   JobInfo.Builder builder = new JobInfo.Builder(jobId, serviceComponent)
       .setRequiredNetworkType(NetworkType.UNMETERED)
       .setRequiresCharging(true)
       .setRequiresDeviceIdle(true)
       .setPersisted(true);
   ```

3. **延迟容忍**：
   - `setMinimumLatency()`: 最小延迟
   - `setOverrideDeadline()`: 最大延迟
   - 让系统选择最优时机

**与Doze协同**：
- Doze期间任务延迟
- Maintenance window执行
- 紧急任务使用`setImportantWhileForeground()`

**最佳实践**：
- 避免精确时间要求
- 合理设置约束条件
- 使用WorkManager统一API

#### WakeLock管理

WakeLock防止设备休眠，需要谨慎使用：

**WakeLock类型**：
1. **PARTIAL_WAKE_LOCK**：
   - CPU保持运行
   - 屏幕和键盘灯可关闭

2. **SCREEN_DIM_WAKE_LOCK**：
   - CPU、屏幕开启
   - 屏幕变暗

3. **SCREEN_BRIGHT_WAKE_LOCK**：
   - CPU、屏幕全亮

4. **FULL_WAKE_LOCK**：
   - CPU、屏幕、键盘全开

**使用原则**：
```java
PowerManager pm = (PowerManager) getSystemService(Context.POWER_SERVICE);
WakeLock wl = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "MyApp:MyWakelockTag");
wl.acquire(timeout); // 总是设置超时
try {
    // 执行工作
} finally {
    wl.release(); // 确保释放
}
```

**系统优化**：
- WakeLock合并
- 自动超时机制
- Doze期间忽略
- 电池统计追踪

**调试工具**：
```bash
# 查看WakeLock统计
adb shell dumpsys power | grep -i wake

# 查看持有者
adb shell dumpsys power | grep "Wake Locks"
```

### 4.3 硬件协同优化

功耗优化需要软硬件协同工作。

#### GPU DVFS

GPU动态电压频率调节(DVFS)类似CPU调频：

**GPU Governor**：
1. **simple_ondemand**：
   - 基于GPU负载
   - 简单有效

2. **msm-adreno-tz**：
   - 高通方案
   - Trustzone辅助

3. **mali_dvfs**：
   - ARM Mali方案
   - 多级频率

**调优参数**：
```bash
# GPU频率查看
cat /sys/class/kgsl/kgsl-3d0/devfreq/cur_freq

# 可用频率
cat /sys/class/kgsl/kgsl-3d0/devfreq/available_frequencies

# GPU负载
cat /sys/class/kgsl/kgsl-3d0/gpu_busy_percentage
```

**优化策略**：
- 根据渲染复杂度调频
- 考虑thermal限制
- VSync对齐
- 与CPU协同

#### Display功耗优化

显示屏是最大的功耗组件之一：

**硬件特性**：
1. **OLED优化**：
   - 黑色像素不耗电
   - 支持Dark mode
   - AOD (Always On Display)

2. **刷新率调节**：
   - 自适应刷新率
   - 静态内容降频
   - 视频匹配帧率

3. **亮度管理**：
   - 自动亮度算法
   - 环境光感应
   - 内容自适应

**软件优化**：
- HWC (Hardware Composer)优化
- 减少过度绘制
- Surface压缩
- Partial update

**LTPO技术**：
```java
// 动态刷新率
Display.Mode[] modes = display.getSupportedModes();
displayManager.setDesiredDisplayModeSpecs(
    displayId, 
    new Display.Mode[]{targetMode}
);
```

#### 5G modem省电

5G带来更大的功耗挑战：

**省电技术**：
1. **EN-DC优化**：
   - 4G/5G智能切换
   - 仅数据业务用5G

2. **天线功耗**：
   - 动态天线调谐
   - MIMO配置优化

3. **DRX (非连续接收)**：
   - 周期性休眠
   - 快速唤醒

**软件策略**：
- 应用网络质量感知
- 后台限制5G
- 智能预取
- 连接聚合

#### AI加速器功耗控制

NPU/DSP等AI加速器的功耗管理：

**功耗特点**：
- 瞬时功耗高
- 任务突发性
- 与CPU/GPU协同

**优化方法**：
1. **任务调度**：
   - 批处理推理
   - 负载均衡
   - 优先级管理

2. **精度优化**：
   - INT8量化
   - 模型压缩
   - 动态精度

3. **硬件控制**：
   - 动态电压调节
   - 核心开关
   - 内存带宽控制

**框架集成**：
```java
// NNAPI功耗提示
ANeuralNetworksCompilation_setPreference(
    compilation,
    ANEURALNETWORKS_PREFER_LOW_POWER
);
```

**监控工具**：
- 专用功耗计数器
- 温度监控
- 利用率统计

## 本章小结

本章深入探讨了Android系统的实时性保证和性能优化策略。我们从Linux内核调度器的Android定制开始，了解了RT调度、EAS能效调度、大小核架构等关键技术。在Jank检测与优化部分，我们分析了Android图形渲染管线、VSYNC机制、Choreographer调度，以及使用Systrace/Perfetto进行性能分析的方法。内存压力处理章节介绍了从LMK到LMKD的演进、PSI压力监控、内存回收策略以及应用层内存优化技巧。最后，我们探讨了从CPU、系统到硬件层面的全方位功耗优化策略。

关键要点：
- RT调度器为关键任务提供确定性保证，音频、触摸、渲染等子系统依赖RT优先级
- Jank产生有多种原因，需要系统性分析和优化，Perfetto提供了强大的分析能力
- 内存管理从内核空间迁移到用户空间带来更大灵活性，PSI提供了更准确的压力信号
- 功耗优化需要软硬件协同，从调度器、框架到硬件加速器的全栈优化

与其他系统对比：
- iOS的GCD和QoS提供了不同的任务优先级模型，更倾向于开发者显式控制
- 鸿蒙的分布式软总线需要考虑跨设备的实时性保证
- Android的开放性带来了更多的优化挑战，但也提供了更大的灵活性

## 练习题

### 基础题

1. **RT调度理解**
   解释Android中RT调度类与CFS调度类的主要区别，并列举至少3个使用RT调度的系统组件。
   
   <details>
   <summary>提示</summary>
   考虑调度策略、优先级范围、抢占规则等方面的差异。
   </details>
   
   <details>
   <summary>参考答案</summary>
   
   主要区别：
   - 调度策略：RT使用SCHED_FIFO/SCHED_RR，CFS使用SCHED_NORMAL
   - 优先级：RT优先级1-99，数值越大优先级越高；CFS使用nice值-20到19
   - 抢占规则：RT任务总是抢占CFS任务，高优先级RT抢占低优先级
   - 时间片：RT可以无限运行（受限于rt_runtime），CFS基于虚拟运行时间公平分配
   
   使用RT调度的组件：
   - AudioFlinger (音频混音线程)
   - InputDispatcher (触摸事件分发)
   - SurfaceFlinger (关键渲染路径)
   - Camera HAL (预览线程)
   - FastMixer (低延迟音频)
   </details>

2. **Jank检测方法**
   描述如何使用dumpsys gfxinfo命令分析应用的渲染性能，解释输出中的关键指标。
   
   <details>
   <summary>提示</summary>
   关注帧时间统计、各阶段耗时、janky帧比例等。
   </details>
   
   <details>
   <summary>参考答案</summary>
   
   使用方法：
   ```bash
   adb shell dumpsys gfxinfo <package_name>
   ```
   
   关键指标：
   - Total frames rendered: 总渲染帧数
   - Janky frames: 超过阈值的帧数和百分比
   - 50th/90th/95th/99th percentile: 帧时间分布
   - Frame time各阶段：Input、Animation、Measure、Layout、Draw、Sync、Command、Swap
   - Number Missed Vsync: 错过的VSYNC数
   - Number High input latency: 高输入延迟次数
   
   通过这些指标可以识别性能瓶颈在哪个阶段。
   </details>

3. **内存压力信号**
   比较vmpressure和PSI两种内存压力监控机制的优缺点。
   
   <details>
   <summary>提示</summary>
   考虑准确性、开销、信息粒度等方面。
   </details>
   
   <details>
   <summary>参考答案</summary>
   
   vmpressure：
   - 优点：实现简单，开销小，长期稳定
   - 缺点：粒度粗，只有low/medium/critical三级，可能误报
   
   PSI (Pressure Stall Information)：
   - 优点：更准确反映实际压力，区分some/full压力，提供时间窗口平均值
   - 缺点：需要Linux 4.20+，计算开销稍大
   
   PSI通过追踪任务因资源等待而阻塞的时间，提供了更准确的压力指标，Android Q开始优先使用PSI。
   </details>

4. **功耗优化基础**
   列举Doze模式的进入条件和主要限制，说明Light Doze与Deep Doze的区别。
   
   <details>
   <summary>提示</summary>
   考虑设备状态、网络访问、唤醒锁等方面。
   </details>
   
   <details>
   <summary>参考答案</summary>
   
   Doze进入条件：
   - 屏幕关闭
   - 电池供电（未充电）
   - 设备静止（加速度计检测）
   - 一段时间无用户交互
   
   主要限制：
   - 网络访问暂停（除高优先级FCM）
   - WakeLock被忽略
   - 标准Alarm延迟（除setAlarmClock）
   - WiFi扫描停止
   - 同步和作业延迟执行
   
   Light Doze vs Deep Doze：
   - Light：设备在口袋中移动时触发，保持网络连接，限制较宽松
   - Deep：设备完全静止时触发，断开网络，进入深度省电
   </details>

### 挑战题

5. **调度器优化方案**
   设计一个优化方案，解决游戏应用在复杂场景下的卡顿问题。需要考虑CPU调度、GPU渲染、内存管理等多个方面。
   
   <details>
   <summary>提示</summary>
   从游戏线程优先级、CPU亲和性、内存预分配、thermal管理等角度思考。
   </details>
   
   <details>
   <summary>参考答案</summary>
   
   综合优化方案：
   
   1. CPU调度优化：
   - 游戏主线程设置为SCHED_FIFO，RT优先级80-85
   - 渲染线程绑定到大核，使用CPU亲和性
   - 启用Game Mode，触发CPU boost
   - 禁用小核或限制后台任务到小核
   
   2. 内存管理：
   - 预分配内存池，避免运行时分配
   - 调高oom_adj分数，减少被杀概率
   - 使用mlockall()锁定关键内存页
   - 提前加载资源，避免运行时IO
   
   3. GPU优化：
   - 根据场景复杂度动态调整渲染分辨率
   - 使用VRS (Variable Rate Shading)
   - 优化Draw Call批处理
   - 监控GPU温度，动态调整画质
   
   4. 系统协同：
   - 请求性能模式，禁用省电特性
   - 使用Sustained Performance Mode
   - 监控thermal状态，提前降低负载
   - 与厂商游戏加速框架集成
   </details>

6. **内存泄漏诊断**
   一个社交应用在后台运行数小时后被系统频繁杀死，设计完整的诊断和优化流程。
   
   <details>
   <summary>提示</summary>
   考虑内存泄漏检测、后台任务优化、内存占用分析等。
   </details>
   
   <details>
   <summary>参考答案</summary>
   
   诊断流程：
   
   1. 初步分析：
   ```bash
   # 查看内存信息
   adb shell dumpsys meminfo <package>
   # 查看oom_adj
   adb shell cat /proc/<pid>/oom_adj
   # 查看内存压力
   adb shell cat /proc/pressure/memory
   ```
   
   2. 泄漏检测：
   - 集成LeakCanary监控泄漏
   - 使用Android Studio Profiler录制内存
   - 分析heap dump找到泄漏对象
   - 检查静态引用、Handler、监听器等
   
   3. 后台优化：
   - 将Service改为JobIntentService
   - 使用WorkManager替代AlarmManager
   - 及时释放不需要的资源
   - 实现onTrimMemory()响应内存压力
   
   4. 内存优化：
   - 优化图片缓存大小和策略
   - 使用更高效的数据结构
   - 减少内存中的数据冗余
   - Native内存及时释放
   
   5. 监控方案：
   - 建立内存使用基准线
   - 监控关键场景内存增长
   - 设置内存告警阈值
   - 定期进行内存回归测试
   </details>

7. **跨平台性能对比**
   分析Android、iOS和鸿蒙在处理高帧率(120Hz)游戏时的架构差异和优化策略。
   
   <details>
   <summary>提示</summary>
   从渲染管线、调度策略、功耗管理等角度对比。
   </details>
   
   <details>
   <summary>参考答案</summary>
   
   架构对比：
   
   Android:
   - Choreographer基于VSYNC的帧调度
   - Triple Buffering减少掉帧
   - RenderThread独立渲染
   - 支持Variable Refresh Rate
   - Vulkan/OpenGL ES渲染
   
   iOS:
   - CADisplayLink驱动的渲染循环
   - Metal渲染，更低开销
   - ProMotion自适应刷新率
   - 统一内存架构，减少拷贝
   - 更严格的后台限制
   
   鸿蒙:
   - 图形图像子系统支持分布式渲染
   - ArkUI声明式UI框架
   - 基于Vulkan的统一渲染
   - 智能调度考虑多设备协同
   - 方舟编译器优化
   
   优化策略差异：
   - Android：开放但碎片化，需要适配更多硬件
   - iOS：封闭优化，硬件软件深度集成
   - 鸿蒙：分布式场景优化，跨设备体验一致性
   </details>

8. **实时音频处理优化**
   设计一个专业音频应用的低延迟处理方案，要求往返延迟小于10ms。
   
   <details>
   <summary>提示</summary>
   考虑音频HAL、缓冲区大小、线程优先级、CPU亲和性等。
   </details>
   
   <details>
   <summary>参考答案</summary>
   
   低延迟方案设计：
   
   1. 音频路径优化：
   - 使用AAudio的EXCLUSIVE模式
   - 设置PERFORMANCE模式
   - 最小缓冲区配置（2-3个period）
   - 采样率匹配硬件原生率
   
   2. 线程配置：
   ```c
   // 音频回调线程
   pthread_setschedparam(thread, SCHED_FIFO, 95);
   // CPU亲和性
   cpu_set_t cpuset;
   CPU_SET(6, &cpuset); // 绑定到大核
   pthread_setaffinity_np(thread, sizeof(cpuset), &cpuset);
   ```
   
   3. 内存优化：
   - 预分配所有音频缓冲区
   - 使用lock-free环形缓冲区
   - mlockall()锁定内存
   - 避免动态内存分配
   
   4. 系统配置：
   - 关闭CPU频率调节
   - 禁用C-states深度休眠
   - 提高中断亲和性优先级
   - 使用RT throttling保护
   
   5. 延迟测量：
   - 硬件loopback测试
   - 时间戳精确测量
   - 分析各阶段延迟
   - 持续监控性能指标
   </details>

## 常见陷阱与错误

### 调度器相关

1. **RT优先级滥用**
   - 错误：给所有线程设置RT优先级
   - 后果：系统响应变差，可能导致看门狗超时
   - 正确：仅关键实时任务使用RT，合理分配优先级

2. **CPU亲和性误用**
   - 错误：将所有线程绑定到大核
   - 后果：大核过载，小核空闲，功耗增加
   - 正确：根据任务特性分配，考虑热平衡

### 性能分析错误

3. **Systrace数据误读**
   - 错误：只看单帧数据下结论
   - 问题：偶发问题被忽略
   - 正确：收集足够样本，分析分布情况

4. **过度优化**
   - 错误：为了性能牺牲所有功能
   - 问题：用户体验下降
   - 正确：平衡性能和功能，数据驱动优化

### 内存管理陷阱

5. **内存泄漏忽视**
   - 错误：认为GC会处理一切
   - 后果：应用被频繁杀死
   - 正确：主动检测和修复泄漏

6. **Native内存失控**
   - 错误：只关注Java堆
   - 问题：Native内存超限导致OOM
   - 正确：全面监控Java堆和Native内存

### 功耗优化误区

7. **WakeLock泄漏**
   - 错误：acquire后忘记release
   - 后果：电池快速耗尽
   - 正确：使用try-finally确保释放

8. **忽视后台限制**
   - 错误：假设后台可以自由运行
   - 问题：被系统限制或杀死
   - 正确：适配Doze和App Standby

## 最佳实践检查清单

### 设计阶段
- [ ] 识别关键性能路径和实时性要求
- [ ] 设计合理的线程模型和优先级方案
- [ ] 规划内存使用预算和缓存策略
- [ ] 考虑功耗影响，设计省电方案
- [ ] 预留性能监控和分析接口

### 开发阶段
- [ ] 使用适当的调度策略和优先级
- [ ] 实现高效的内存管理和复用
- [ ] 避免主线程阻塞操作
- [ ] 合理使用系统资源（WakeLock、网络等）
- [ ] 添加性能追踪点（Trace.beginSection）

### 测试阶段
- [ ] 使用Systrace/Perfetto分析性能
- [ ] 进行内存泄漏检测（LeakCanary）
- [ ] 测试不同内存压力下的表现
- [ ] 验证Doze模式下的行为
- [ ] 进行功耗测试和优化

### 优化阶段
- [ ] 基于数据识别性能瓶颈
- [ ] 优化关键路径的执行效率
- [ ] 减少内存分配和GC压力
- [ ] 实现自适应性能策略
- [ ] 持续监控线上性能指标

### 维护阶段
- [ ] 建立性能基准线和告警
- [ ] 定期进行性能回归测试
- [ ] 跟踪新版本系统的优化特性
- [ ] 收集用户反馈持续改进
- [ ] 更新文档和最佳实践
