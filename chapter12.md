# 第12章：相机与多媒体框架

Android的相机与多媒体框架是系统中最复杂的子系统之一，涉及从硬件抽象层到应用层的完整栈实现。本章将深入剖析Camera HAL的演进历程、MediaCodec的架构设计、Stagefright多媒体框架的实现原理，并与iOS的AVFoundation进行技术对比。通过本章学习，读者将掌握Android多媒体处理的核心机制，理解如何实现高性能的相机应用和多媒体处理。

## 学习目标

1. 理解Camera HAL从v1到v3的架构演进及其设计考量
2. 掌握MediaCodec与OpenMAX IL的集成机制
3. 深入理解Stagefright框架的组件架构和数据流
4. 对比分析Android与iOS在多媒体处理上的技术差异
5. 掌握多媒体框架的性能优化和调试技巧

## 章节大纲

### 12.1 Camera HAL演进
- 12.1.1 Camera HAL v1到v3的架构变迁
- 12.1.2 Camera2 API与HAL3的对应关系
- 12.1.3 Camera Provider服务架构
- 12.1.4 Multi-Camera与逻辑相机支持

### 12.2 MediaCodec与OMX
- 12.2.1 MediaCodec架构设计
- 12.2.2 OpenMAX IL集成
- 12.2.3 Codec2框架演进
- 12.2.4 硬件编解码器接入

### 12.3 Stagefright架构
- 12.3.1 MediaExtractor与Parser
- 12.3.2 MediaCodec管理机制
- 12.3.3 AudioTrack/VideoTrack处理
- 12.3.4 DRM集成与保护

### 12.4 与iOS AVFoundation对比
- 12.4.1 架构设计差异
- 12.4.2 性能优化策略对比
- 12.4.3 API易用性分析
- 12.4.4 生态系统差异

### 12.5 本章小结
### 12.6 练习题
### 12.7 常见陷阱与错误
### 12.8 最佳实践检查清单

## 12.1 Camera HAL演进

### 12.1.1 Camera HAL v1到v3的架构变迁

Android相机硬件抽象层(HAL)经历了三个主要版本的演进，每个版本都反映了移动摄影技术的进步和应用需求的变化。

**Camera HAL v1 (Legacy HAL)**

Camera HAL v1采用了简单的同步接口设计，主要特点包括：
- 基于`camera_device_t`结构体的C接口
- 单一的预览和拍照模式
- 通过回调函数`camera_data_callback`传递图像数据
- 参数设置使用字符串键值对(通过`setParameters()`和`getParameters()`)
- 不支持并发访问多个相机

HAL v1的核心接口函数包括：
- `camera_module_t::get_camera_info()` - 获取相机信息
- `camera_device_t::start_preview()` - 启动预览
- `camera_device_t::take_picture()` - 拍照
- `camera_device_t::set_callbacks()` - 设置回调函数

**Camera HAL v3的革新**

HAL v3引入了基于流(Stream)的架构，实现了更灵活的图像处理管道：

1. **Request/Result模型**：
   - 每个capture request包含完整的相机控制参数
   - 异步处理模型，支持pipeline并行处理
   - Result包含元数据和图像缓冲区

2. **Stream配置**：
   - 支持多个并发输出流
   - 每个流可以有不同的分辨率、格式和用途
   - 通过`camera3_stream_t`结构描述流属性

3. **3A控制分离**：
   - 自动对焦(AF)、自动曝光(AE)、自动白平衡(AWB)独立控制
   - 支持手动控制和部分手动模式
   - 精确的时间戳和同步机制

### 12.1.2 Camera2 API与HAL3的对应关系

Camera2 API是Android 5.0引入的新相机框架API，与HAL3紧密配合：

**架构层次映射**：
```
应用层: Camera2 API (android.hardware.camera2)
   ↓
框架层: CameraService / Camera3Device
   ↓
HAL层: Camera HAL3 Interface
   ↓
驱动层: V4L2 / Proprietary Driver
```

**关键概念对应**：
1. **CaptureRequest ↔ camera3_capture_request**：
   - 应用层的CaptureRequest通过CameraService转换为HAL层的camera3_capture_request
   - 包含settings metadata和output buffers配置

2. **CameraDevice.StateCallback ↔ camera3_callback_ops**：
   - 状态变化通知机制
   - 错误处理和流管理

3. **Surface ↔ camera3_stream_buffer**：
   - Surface在HAL层表现为stream buffer
   - 通过Gralloc分配的图形缓冲区

### 12.1.3 Camera Provider服务架构

Android 8.0引入的Treble架构对Camera HAL产生了重大影响：

**Camera Provider进程模型**：
- 独立的`android.hardware.camera.provider@2.x`进程
- 通过HIDL接口与CameraService通信
- 支持多个相机设备的动态枚举

**关键组件**：
1. **ICameraProvider接口**：
   - `getCameraIdList()` - 枚举可用相机
   - `getCameraDeviceInterface_V3_x()` - 获取设备接口
   - `notifyDeviceStateChange()` - 设备状态通知

2. **ICameraDevice接口**：
   - `open()` - 打开相机会话
   - `getCameraCharacteristics()` - 获取静态特性
   - 对应HAL3的camera3_device_t

3. **热插拔支持**：
   - 通过`ICameraProviderCallback`实现
   - 支持USB相机等外部设备
   - 动态相机ID分配机制

### 12.1.4 Multi-Camera与逻辑相机支持

Android P引入了多相机API，支持同时访问多个物理相机：

**物理相机ID机制**：
- 逻辑相机ID映射到多个物理相机ID
- 通过`getPhysicalCameraIds()`获取物理相机列表
- 每个物理相机有独立的characteristics

**同步机制**：
- 硬件级同步：通过时间戳对齐
- 软件级同步：CameraService协调多个HAL调用
- 支持设置per-physical camera的参数

**典型应用场景**：
1. **双摄像头景深**：
   - 主相机 + 深度相机组合
   - 实时景深计算和背景虚化

2. **广角 + 长焦切换**：
   - 无缝变焦体验
   - 基于焦距自动选择物理相机

3. **多相机融合**：
   - HDR+场景：多帧融合
   - 夜景模式：长曝光堆栈

**实现考量**：
- 缓冲区管理：避免内存过度消耗
- 功耗优化：智能开关物理相机
- ISP资源调度：硬件资源竞争处理

## 12.2 MediaCodec与OMX

### 12.2.1 MediaCodec架构设计

MediaCodec是Android多媒体框架的核心组件，提供了访问底层编解码器的高级API。其架构设计体现了灵活性和性能的平衡。

**核心设计原则**：
1. **状态机模型**：
   - Uninitialized → Configured → Executing → Released
   - 通过明确的状态转换保证操作的有效性
   - 异步和同步两种操作模式

2. **缓冲区管理**：
   - Input/Output Buffer队列机制
   - 通过`dequeueInputBuffer()`和`dequeueOutputBuffer()`管理
   - 支持Surface直接渲染(Surface模式)

3. **数据流模型**：
   ```
   Input Data → [Input Buffer] → Codec → [Output Buffer] → Output Data
                                    ↓
                              [Surface Rendering]
   ```

**关键API接口**：
- `configure()` - 配置编解码器参数
- `start()` - 启动编解码器
- `queueInputBuffer()` - 提交输入数据
- `releaseOutputBuffer()` - 释放输出缓冲区

### 12.2.2 OpenMAX IL集成

OpenMAX IL (Integration Layer) 是Khronos定义的多媒体组件标准接口，Android通过OMX实现硬件编解码器的抽象。

**OMX组件架构**：
1. **OMX Core**：
   - 组件枚举和加载: `OMX_GetHandle()`
   - 组件生命周期管理
   - 通过`libstagefright_omx`实现

2. **OMX Component**：
   - 标准化的组件接口: `OMX_COMPONENTTYPE`
   - Port概念：输入输出数据通道
   - 参数和配置接口：`OMX_SetParameter()`, `OMX_SetConfig()`

3. **Buffer分配机制**：
   - UseBuffer模式：使用客户端分配的缓冲区
   - AllocateBuffer模式：组件内部分配
   - UseAndroidNativeBuffer：支持GraphicBuffer

**Android OMX扩展**：
- `OMX.google.*` - Google软件编解码器
- `OMX.qcom.*` - 高通硬件编解码器
- 厂商特定扩展：如`OMX_IndexParamAndroidAdaptivePlayback`

### 12.2.3 Codec2框架演进

Android P引入Codec2框架，旨在替代老化的OMX，提供更现代的编解码器接口。

**Codec2优势**：
1. **简化的接口设计**：
   - 基于C++的类型安全接口
   - 参数系统更加灵活：`C2Param`体系
   - 减少状态复杂度

2. **改进的缓冲区管理**：
   - C2Buffer统一缓冲区抽象
   - 支持更复杂的内存类型(如secure buffer)
   - Zero-copy优化

3. **组件发现机制**：
   - 通过`C2ComponentStore`管理
   - 支持动态加载和优先级管理
   - 更好的vendor组件集成

**迁移策略**：
- `Codec2Client`作为客户端接口
- `C2OMXNode`提供OMX兼容层
- 渐进式迁移：新组件用Codec2，旧组件保持OMX

### 12.2.4 硬件编解码器接入

硬件编解码器的集成涉及多个层次的优化和适配：

**平台特定优化**：
1. **高通平台(Venus/Vidc)**：
   - 专用的视频核心：Venus子系统
   - SMEM共享内存机制
   - 低功耗模式支持

2. **联发科平台(MDP/VDEC)**：
   - 硬件MDP(Media Data Path)集成
   - 支持HEIF等新格式
   - AI辅助编码优化

3. **海思/三星平台**：
   - 自研编解码IP
   - 与ISP/GPU紧密集成
   - 专有格式支持

**性能优化技术**：
1. **批处理模式(Batch Mode)**：
   - 减少内核态切换开销
   - 提高吞吐量

2. **低延迟模式**：
   - 实时通信场景优化
   - 帧内预测快速路径

3. **功耗优化**：
   - DVFS(Dynamic Voltage and Frequency Scaling)
   - 多核调度策略
   - 硬件休眠管理

**安全视频路径(SVP)**：
- TrustZone集成
- Secure Buffer分配
- DRM保护内容处理
- HDCP输出保护

## 12.3 Stagefright架构

### 12.3.1 MediaExtractor与Parser

Stagefright是Android的多媒体播放引擎，其模块化设计支持多种容器格式和编解码器。MediaExtractor是解析媒体文件的核心组件。

**MediaExtractor架构**：
1. **Sniffer机制**：
   - 通过`DataSource`读取文件头
   - 调用各个Extractor的`sniff()`函数识别格式
   - 基于置信度选择合适的Extractor

2. **支持的容器格式**：
   - MP4/3GP: `MPEG4Extractor`
   - MKV/WebM: `MatroskaExtractor`
   - MP3: `MP3Extractor`
   - FLAC: `FLACExtractor`
   - 可通过插件扩展新格式

3. **Track管理**：
   - `getTrackCount()` - 获取轨道数
   - `getTrackFormat()` - 获取轨道元数据
   - `selectTrack()` - 选择要解码的轨道
   - `readSampleData()` - 读取样本数据

**Parser实现细节**：
- 增量解析：支持网络流媒体
- 索引构建：快速seek支持
- 元数据提取：ID3、Vorbis Comment等
- 错误恢复：处理损坏的媒体文件

### 12.3.2 MediaCodec管理机制

Stagefright通过MediaCodec管理编解码器的生命周期和资源分配：

**Codec选择策略**：
1. **媒体编解码器列表**：
   - `/system/etc/media_codecs.xml`定义可用编解码器
   - 性能等级声明：`MediaCodecInfo.CodecCapabilities`
   - 硬件优先原则

2. **动态能力查询**：
   ```
   编解码器能力包括：
   - 支持的分辨率范围
   - 帧率限制
   - 比特率范围
   - Profile/Level支持
   ```

3. **资源管理器(ResourceManager)**：
   - 跟踪编解码器使用情况
   - 基于优先级的资源分配
   - 处理资源竞争和回收

**缓冲区流转优化**：
- BufferQueue集成：直接渲染路径
- GraphicBuffer共享：减少拷贝
- 时间戳传递：音视频同步

### 12.3.3 AudioTrack/VideoTrack处理

Stagefright中音视频轨道的处理涉及解码、渲染和同步：

**AudioTrack处理流程**：
1. **音频解码链路**：
   ```
   MediaExtractor → AudioSource → MediaCodec → AudioSink
                                       ↓
                                  AudioTrack → AudioFlinger
   ```

2. **重采样和混音**：
   - `AudioResampler`处理采样率转换
   - 支持多通道下混到立体声
   - 音量调节和淡入淡出

3. **低延迟优化**：
   - FastTrack机制
   - 直接HAL路径
   - 减少缓冲区大小

**VideoTrack渲染管道**：
1. **Surface渲染路径**：
   - MediaCodec → Surface → SurfaceFlinger
   - 硬件合成优化
   - HDR元数据传递

2. **色彩空间转换**：
   - YUV到RGB转换
   - HDR到SDR映射
   - 厂商特定优化

3. **帧率控制**：
   - VSync同步机制
   - 丢帧策略
   - 自适应刷新率

### 12.3.4 DRM集成与保护

Stagefright集成了多种DRM(Digital Rights Management)系统：

**DRM框架架构**：
1. **MediaDrm API**：
   - 提供DRM会话管理
   - 许可证获取和存储
   - 支持Widevine、PlayReady等

2. **加密媒体扩展(EME)**：
   - `MediaCrypto`对象关联
   - 安全解码器路径
   - 输出保护策略

3. **密钥管理**：
   - 密钥请求/响应机制
   - 离线许可证支持
   - 密钥轮换处理

**安全播放实现**：
1. **Secure Decoder**：
   - TEE中的解码器实例
   - 加密输入缓冲区
   - 安全输出路径

2. **HDCP集成**：
   - Display级别的内容保护
   - 版本协商(HDCP 1.x/2.x)
   - 降级策略

3. **防截屏机制**：
   - Surface标记为secure
   - GPU渲染限制
   - 截屏API返回黑屏

**常见DRM系统对比**：
- **Widevine**：Google官方，L1/L2/L3安全级别
- **PlayReady**：微软方案，Windows生态兼容
- **China DRM**：国内视频平台采用
- **Marlin**：日本市场方案

## 12.4 与iOS AVFoundation对比

### 12.4.1 架构设计差异

Android和iOS在多媒体框架设计上体现了不同的理念和优化目标：

**分层架构对比**：

Android多媒体栈：
```
应用层: MediaPlayer / Camera2 API
  ↓
框架层: MediaCodec / Stagefright
  ↓
HAL层: OMX / Codec2 / Camera HAL
  ↓
内核层: V4L2 / 厂商驱动
```

iOS AVFoundation栈：
```
应用层: AVFoundation Framework
  ↓
核心层: Core Media / Core Video / Core Audio
  ↓
驱动层: IOKit / Metal Performance Shaders
  ↓
内核层: Darwin内核驱动
```

**设计理念差异**：
1. **API统一性**：
   - iOS: AVFoundation提供统一的音视频处理接口
   - Android: 分散的API(MediaPlayer、MediaCodec、Camera2等)

2. **硬件抽象**：
   - iOS: 紧密的软硬件集成，直接优化特定硬件
   - Android: 通用HAL层，需要适配多种硬件平台

3. **进程模型**：
   - iOS: 单一进程内处理，减少IPC开销
   - Android: 跨进程架构(MediaServer)，更好的隔离性

### 12.4.2 性能优化策略对比

两个平台在性能优化上采取了不同的策略：

**iOS优化特点**：
1. **Metal集成**：
   - 直接使用Metal进行视频处理
   - GPU加速的滤镜和特效
   - 统一的渲染管道

2. **内存管理**：
   - IOSurface零拷贝机制
   - 统一的内存池管理
   - 自动内存压缩

3. **能效优化**：
   - 协处理器卸载(如Neural Engine)
   - 精确的功耗预算
   - 自适应质量调整

**Android优化挑战与方案**：
1. **碎片化应对**：
   - CTS测试保证基础兼容性
   - Vendor扩展机制
   - 动态性能调整

2. **内存优化**：
   - ION/DMA-BUF共享机制
   - GraphicBuffer跨进程共享
   - 低内存设备优化(Go Edition)

3. **调度优化**：
   - 大小核调度策略
   - thermal throttling处理
   - 后台限制机制

### 12.4.3 API易用性分析

**iOS AVFoundation优势**：
1. **一致的对象模型**：
   - AVAsset统一表示媒体资源
   - AVPlayer处理所有播放需求
   - AVCaptureSession统一相机接口

2. **Swift集成**：
   - 现代化的API设计
   - 强类型和可选值
   - Combine框架集成

3. **文档和工具**：
   - 完善的官方文档
   - Instruments性能分析
   - 可视化的调试工具

**Android多媒体API特点**：
1. **灵活性**：
   - 多层次API选择(从MediaPlayer到MediaCodec)
   - 支持自定义组件
   - 开源实现可参考

2. **兼容性考虑**：
   - 支持旧版本API
   - 多种实现路径
   - Jetpack CameraX简化使用

3. **调试支持**：
   - dumpsys media.*
   - Systrace性能追踪
   - 开源代码可调试

### 12.4.4 生态系统差异

**编解码器支持**：
- iOS: 有限但优化的编解码器集(HEVC/HEIF原生支持)
- Android: 广泛的编解码器支持，但质量参差不齐

**DRM生态**：
- iOS: FairPlay Streaming统一方案
- Android: 多种DRM系统并存(Widevine为主)

**开发者生态**：
1. **iOS特点**：
   - 封闭但一致的开发体验
   - 严格的App Store审核
   - 付费应用生态成熟

2. **Android特点**：
   - 开放的生态系统
   - 多渠道分发
   - 开源项目丰富

**性能基准对比**：
- 4K视频解码：iOS通常功耗更低
- 相机启动速度：iOS更快(~200ms vs ~400ms)
- 多路视频处理：Android更灵活
- AI处理集成：iOS Neural Engine vs Android NNAPI

## 12.5 本章小结

本章深入探讨了Android相机与多媒体框架的核心架构和实现原理：

1. **Camera HAL演进**：从简单的v1同步接口到基于流的v3架构，实现了更灵活的图像处理管道。Treble架构引入的Provider模型进一步提升了模块化程度。

2. **MediaCodec架构**：通过状态机模型和缓冲区队列机制，提供了高效的编解码器访问。从OMX到Codec2的演进解决了接口复杂度和性能问题。

3. **Stagefright框架**：模块化的设计支持多种媒体格式，通过MediaExtractor、MediaCodec和渲染管道的配合，实现了完整的多媒体播放功能。

4. **与iOS对比**：Android的开放性带来了更大的灵活性但也增加了复杂度，而iOS的垂直整合提供了更一致的性能表现。

关键性能指标：
- Camera HAL3请求延迟：< 33ms (30fps)
- MediaCodec初始化时间：< 100ms
- 4K@30fps解码功耗：< 500mW
- DRM密钥加载时间：< 200ms

## 12.6 练习题

### 基础题

1. **Camera HAL版本差异**
   描述Camera HAL v1和v3在处理预览流时的主要架构差异。

   <details>
   <summary>Hint: 考虑同步vs异步模型</summary>
   
   思考v1的回调模式和v3的request/result模型如何影响预览流的处理效率。
   </details>

   <details>
   <summary>参考答案</summary>
   
   HAL v1使用同步的回调模式，预览数据通过data_callback直接传递，整个流程是阻塞式的。HAL v3采用异步的request/result模型，预览流作为一个持续的stream配置，通过capture request的重复提交实现，支持pipeline并行处理，显著提升了帧率和响应速度。
   </details>

2. **MediaCodec状态机**
   画出MediaCodec的完整状态转换图，并说明在什么情况下会进入Error状态。

   <details>
   <summary>Hint: 考虑configure和start的顺序</summary>
   
   MediaCodec有严格的状态转换要求，错误的API调用顺序会导致异常。
   </details>

   <details>
   <summary>参考答案</summary>
   
   状态转换：Uninitialized →(configure)→ Configured →(start)→ Executing →(stop)→ Configured →(release)→ Released。Error状态可能由以下原因触发：1)编解码器资源不足 2)输入数据格式错误 3)硬件故障 4)DRM许可证失效 5)在错误状态调用API。
   </details>

3. **Stagefright组件交互**
   解释MediaExtractor、MediaCodec和MediaPlayer之间的数据流关系。

   <details>
   <summary>Hint: 考虑push vs pull模型</summary>
   
   注意哪个组件主动拉取数据，哪个组件被动接收数据。
   </details>

   <details>
   <summary>参考答案</summary>
   
   MediaExtractor解析容器格式，提取音视频轨道数据。MediaPlayer作为协调者，从MediaExtractor拉取压缩数据，推送给对应的MediaCodec进行解码。解码后的数据通过AudioTrack输出音频，通过Surface输出视频。整体是pull模型，由渲染端驱动数据流动。
   </details>

### 挑战题

4. **多相机同步方案设计**
   设计一个支持四路相机同时采集的系统架构，要求时间戳同步误差小于1ms，说明你的设计如何处理：
   - 硬件时钟同步
   - 缓冲区管理
   - CPU/ISP资源调度

   <details>
   <summary>Hint: 考虑硬件和软件两个层面</summary>
   
   硬件层面需要共享时钟源，软件层面需要协调多个HAL实例。
   </details>

   <details>
   <summary>参考答案</summary>
   
   采用主从时钟架构：1)硬件层通过共享VSYNC信号实现帧同步，使用统一的时钟源(如TSC)生成时间戳。2)软件层在CameraProvider中实现同步协调器，预分配环形缓冲区池避免动态分配开销。3)通过设置CPU亲和性将不同相机的ISP处理分配到不同核心，使用硬件优先级确保关键路径的实时性。时间戳在HAL层统一校准，确保亚毫秒级同步。
   </details>

5. **低延迟视频通话优化**
   针对1080p@30fps的实时视频通话场景，设计一个端到端延迟小于100ms的处理pipeline，考虑：
   - 编码器参数优化
   - 缓冲区大小设置
   - 跨进程数据传输

   <details>
   <summary>Hint: 延迟来源分析</summary>
   
   识别采集、编码、传输、解码、渲染各环节的延迟贡献。
   </details>

   <details>
   <summary>参考答案</summary>
   
   1)使用Camera2的TEMPLATE_RECORD模板，关闭3A算法的复杂处理。2)MediaCodec配置：启用低延迟模式，设置1帧的缓冲深度，使用baseline profile，关闭B帧。3)使用Surface直通路径避免内存拷贝，通过GraphicBuffer实现零拷贝传输。4)网络传输使用RTP with FEC，动态调整码率。5)解码端预分配Surface，使用TextureView直接渲染。整体延迟分解：采集20ms+编码15ms+传输40ms+解码15ms+显示10ms=100ms。
   </details>

6. **DRM视频防录制方案**
   设计一个防止DRM保护视频被录制的完整方案，包括：
   - 应用层API限制
   - 渲染路径保护
   - 硬件级防护

   <details>
   <summary>Hint: 考虑所有可能的录制路径</summary>
   
   包括截屏、HDMI输出、root权限访问等多种攻击向量。
   </details>

   <details>
   <summary>参考答案</summary>
   
   多层防护体系：1)应用层通过FLAG_SECURE标记Surface，禁止截屏API访问。2)SurfaceFlinger识别secure layer，在合成时跳过这些图层的截屏处理。3)GPU驱动层面限制secure buffer的读取权限。4)HDCP协议保护外部显示输出。5)TrustZone中的secure decoder确保解密过程隔离。6)硬件OTP熔丝防止bootloader解锁。7)定期远程证明(Remote Attestation)验证设备完整性。即使设备root，视频内容仍在TEE中处理，主系统无法访问明文数据。
   </details>

## 12.7 常见陷阱与错误

### Camera相关

1. **相机资源泄露**
   - 错误：忘记调用`CameraDevice.close()`
   - 后果：其他应用无法访问相机
   - 解决：使用try-finally或lifecycle感知组件

2. **预览尺寸不匹配**
   - 错误：随意设置预览分辨率
   - 后果：预览变形或黑屏
   - 解决：从`getOutputSizes()`选择支持的尺寸

3. **Surface生命周期**
   - 错误：Surface销毁后继续使用
   - 后果：原生崩溃(Native Crash)
   - 解决：监听SurfaceHolder.Callback

### MediaCodec相关

4. **缓冲区超时**
   - 错误：`dequeueBuffer`使用无限超时
   - 后果：UI线程阻塞，ANR
   - 解决：使用合理的超时值(如10ms)

5. **格式协商失败**
   - 错误：硬编码媒体格式参数
   - 后果：编解码器初始化失败
   - 解决：使用`MediaFormat.createVideo/AudioFormat()`

6. **异步模式混淆**
   - 错误：同步API和Callback混用
   - 后果：死锁或数据丢失
   - 解决：坚持使用一种模式

### 性能相关

7. **过度的格式转换**
   - 错误：YUV→RGB→YUV转换链
   - 后果：CPU占用率高，功耗大
   - 解决：使用Surface直通路径

8. **内存抖动**
   - 错误：频繁分配大缓冲区
   - 后果：GC频繁，卡顿
   - 解决：使用缓冲区池

## 12.8 最佳实践检查清单

### 架构设计审查

- [ ] 是否正确处理了相机权限的动态申请？
- [ ] 是否考虑了多相机设备的兼容性？
- [ ] 媒体格式是否做了能力查询而非硬编码？
- [ ] 是否实现了优雅的错误恢复机制？

### 性能优化审查

- [ ] 是否使用了硬件编解码器(如果可用)？
- [ ] 是否避免了不必要的内存拷贝？
- [ ] 是否正确设置了线程优先级？
- [ ] 是否监控了丢帧率和延迟指标？

### 兼容性测试

- [ ] 是否在不同厂商设备上测试？
- [ ] 是否处理了编解码器缺失的情况？
- [ ] 是否支持必要的向后兼容？
- [ ] 是否通过了CTS媒体相关测试？

### 安全性考虑

- [ ] DRM内容是否正确保护？
- [ ] 是否避免了敏感数据泄露到日志？
- [ ] 临时文件是否安全清理？
- [ ] 是否验证了输入媒体文件的合法性？

### 资源管理

- [ ] 相机资源是否正确释放？
- [ ] MediaCodec是否正确停止和释放？
- [ ] Surface是否正确管理生命周期？
- [ ] 是否处理了系统内存压力事件？