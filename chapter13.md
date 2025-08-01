# 第13章：Android安全模型

Android系统采用多层次、纵深防御的安全架构，从Linux内核到应用层都实施了严格的安全控制。本章将深入剖析Android的安全机制，包括应用沙箱、权限系统、SELinux强制访问控制等核心组件，并与iOS等系统进行技术对比，帮助读者理解移动操作系统安全设计的最佳实践。

## 13.1 应用沙箱机制

### 13.1.1 UID隔离基础

Android继承了Linux的多用户机制，但创新性地将其用于应用隔离。每个应用在安装时都会分配唯一的UID（用户ID），通常从10000开始递增。这种设计使得每个应用都运行在独立的Linux用户空间中。

关键实现细节：
- PackageManagerService在应用安装时通过`mSettings.addUserIdLPw()`分配UID
- 系统应用使用预定义的UID（如system uid=1000）
- 共享UID机制允许签名相同的应用共享进程和数据

### 13.1.2 进程隔离实现

每个应用进程通过Zygote fork创建，继承了完整的安全上下文：

1. **进程边界**：通过Linux进程隔离确保内存空间独立
2. **文件系统隔离**：应用私有目录（/data/data/package_name）设置为700权限
3. **IPC限制**：Binder驱动在内核层检查UID/PID，确保跨进程通信安全

### 13.1.3 存储沙箱演进

Android存储模型经历了重大变革：

**Android 10之前**：
- 外部存储（/sdcard）采用FUSE文件系统
- 通过READ_EXTERNAL_STORAGE/WRITE_EXTERNAL_STORAGE权限控制

**Android 10+（Scoped Storage）**：
- MediaStore API成为访问共享存储的主要方式
- 应用只能直接访问自己的外部存储目录
- 通过Storage Access Framework访问其他应用文件

**Android 11+增强**：
- MANAGE_EXTERNAL_STORAGE权限仅限特殊应用
- 文件路径访问进一步受限

### 13.1.4 网络沙箱

Android实现了细粒度的网络访问控制：

1. **INTERNET权限**：控制网络套接字创建
2. **网络策略**：通过iptables/netfilter实现UID级别的流量控制
3. **VPN隔离**：per-app VPN功能允许特定应用流量路由

## 13.2 权限系统演进

### 13.2.1 权限模型基础

Android权限系统基于最小权限原则，主要包括：

**权限级别**：
- normal：低风险权限，安装时自动授予
- dangerous：高风险权限，需要用户明确授权
- signature：仅授予签名相同的应用
- signatureOrSystem：系统应用或签名相同的应用

**权限声明与检查**：
- AndroidManifest.xml中通过`<uses-permission>`声明
- 运行时通过`checkPermission()`/`enforcePermission()`检查
- PackageManagerService维护权限授予状态

### 13.2.2 运行时权限（Android 6.0+）

运行时权限彻底改变了危险权限的授予方式：

**实现机制**：
- PermissionController系统应用负责权限UI
- AppOpsService跟踪权限使用情况
- 权限组概念简化用户决策

**关键API**：
- `requestPermissions()`触发权限请求
- `onRequestPermissionsResult()`处理授权结果
- `shouldShowRequestPermissionRationale()`优化用户体验

### 13.2.3 权限使用审计（Android 10+）

Android引入了更透明的权限使用跟踪：

1. **位置权限细分**：
   - ACCESS_FINE_LOCATION：精确位置
   - ACCESS_COARSE_LOCATION：粗略位置
   - ACCESS_BACKGROUND_LOCATION：后台位置（需单独请求）

2. **权限使用指示器**：
   - 状态栏图标显示摄像头/麦克风使用
   - 隐私仪表板展示历史使用记录

### 13.2.4 特殊权限机制

某些敏感操作需要特殊权限：

**系统权限**：
- SYSTEM_ALERT_WINDOW：悬浮窗权限
- WRITE_SETTINGS：修改系统设置
- REQUEST_INSTALL_PACKAGES：安装应用

**设备管理权限**：
- DeviceAdminReceiver：设备管理器API
- DeviceOwner/ProfileOwner：企业管理功能

## 13.3 SELinux策略

### 13.3.1 SELinux在Android中的实现

Android从4.3开始引入SELinux，5.0起全面启用enforcing模式：

**核心概念**：
- 主体（Subject）：进程的安全上下文
- 客体（Object）：文件、套接字等资源
- 策略（Policy）：定义访问规则

**Android特定实现**：
- 基于TE（Type Enforcement）的强制访问控制
- 通过属性（attribute）简化策略管理
- 分离平台策略和厂商策略（Project Treble）

### 13.3.2 策略文件组织

SELinux策略分布在多个位置：

```
/system/etc/selinux/         # 平台策略
/vendor/etc/selinux/         # 厂商策略
/odm/etc/selinux/           # ODM策略
```

关键策略文件：
- file_contexts：文件安全上下文映射
- property_contexts：属性上下文定义
- service_contexts：服务上下文映射
- seapp_contexts：应用进程上下文规则

### 13.3.3 域转换与类型强制

**进程域转换**：
- init进程根据init.rc中的`seclabel`设置域
- Zygote fork的应用进程通过seapp_contexts确定域
- 通过`type_transition`规则实现自动域转换

**文件类型强制**：
- 每个文件都有安全上下文（通过xattr存储）
- 访问控制基于域和类型的allow规则
- neverallow规则防止策略违规

### 13.3.4 Treble与SELinux

Project Treble引入了更严格的SELinux边界：

1. **平台-厂商分离**：
   - 平台进程不能访问厂商文件类型
   - 厂商进程通过HIDL/AIDL访问平台服务

2. **属性隔离**：
   - vendor_属性仅供厂商代码使用
   - 严格的属性访问控制

3. **策略编译**：
   - 编译时合并多个策略源
   - CIL（Common Intermediate Language）作为中间表示

## 13.4 与iOS沙箱对比

### 13.4.1 架构差异

**Android沙箱**：
- 基于Linux UID的进程隔离
- SELinux提供MAC（强制访问控制）
- 相对开放的文件系统访问

**iOS沙箱**：
- 基于BSD的Mandatory Access Control
- Seatbelt沙箱配置文件
- 更严格的文件系统限制

### 13.4.2 权限模型对比

**Android**：
- 细粒度的权限声明
- 运行时权限请求
- 权限组简化管理

**iOS**：
- 更少的权限类型
- 首次使用时请求
- 隐私标签（Privacy Nutrition Labels）

### 13.4.3 进程间通信安全

**Android Binder**：
- UID/PID验证
- SELinux策略控制
- 接口级别的权限检查

**iOS XPC/Mach**：
- Entitlements验证
- Mach端口权限
- 更严格的IPC限制

### 13.4.4 代码签名与完整性

**Android**：
- APK签名验证（v1/v2/v3/v4）
- dm-verity确保系统分区完整性
- 可选的应用完整性验证

**iOS**：
- 强制代码签名
- 运行时代码完整性检查
- 更严格的动态代码限制

## 本章小结

Android安全模型采用纵深防御策略，主要包括：

1. **应用沙箱**：基于Linux UID的进程隔离，配合文件权限和SELinux策略
2. **权限系统**：从安装时权限演进到运行时权限，提供细粒度的访问控制
3. **SELinux**：强制访问控制补充自主访问控制，Treble进一步加强了系统组件隔离
4. **存储安全**：Scoped Storage限制了应用对共享存储的访问

与iOS相比，Android提供了更灵活但相对复杂的安全模型。理解这些机制对于开发安全的Android应用和进行安全研究至关重要。

## 练习题

### 基础题

1. **UID分配机制**  
   Android为每个应用分配UID的范围是什么？系统应用和普通应用的UID有何区别？  
   **Hint**: 查看Process.java中的UID常量定义

   <details>
   <summary>答案</summary>
   
   普通应用UID从10000开始（FIRST_APPLICATION_UID），系统核心服务使用0-9999范围。system进程使用UID 1000（SYSTEM_UID），root为0。共享UID的应用会使用相同的UID值。
   </details>

2. **权限级别识别**  
   给定权限android.permission.CAMERA和android.permission.ACCESS_NETWORK_STATE，分别属于什么级别？这对用户授权有什么影响？  
   **Hint**: 考虑哪些权限需要运行时请求

   <details>
   <summary>答案</summary>
   
   CAMERA是dangerous权限，需要运行时请求用户授权；ACCESS_NETWORK_STATE是normal权限，安装时自动授予。dangerous权限涉及用户隐私或敏感功能，必须明确授权。
   </details>

3. **SELinux上下文理解**  
   解释SELinux上下文u:r:untrusted_app:s0:c512,c768的含义，每部分代表什么？  
   **Hint**: user:role:type:sensitivity:categories

   <details>
   <summary>答案</summary>
   
   u=user（用户），r=role（角色），untrusted_app=type（域类型），s0=sensitivity（敏感度级别），c512,c768=categories（类别，用于MLS/MCS）。这是普通应用进程的典型安全上下文。
   </details>

4. **存储访问演进**  
   列举Android 10 Scoped Storage相比传统存储模型的三个主要变化。  
   **Hint**: 考虑直接路径访问、权限需求和API变化

   <details>
   <summary>答案</summary>
   
   1) 应用不能直接访问外部存储任意路径 2) 访问媒体文件需通过MediaStore API 3) 访问其他应用文件需要通过Storage Access Framework。READ/WRITE_EXTERNAL_STORAGE权限作用大幅降低。
   </details>

### 挑战题

5. **沙箱逃逸分析**  
   设计一个理论上的Android应用沙箱逃逸场景，需要绕过哪些安全机制？现代Android如何防御此类攻击？  
   **Hint**: 考虑多层防御体系

   <details>
   <summary>答案</summary>
   
   理论逃逸需要：1) 内核漏洞获取root 2) 绕过SELinux策略 3) 突破应用UID限制。防御措施：内核加固(KASLR/PXN/PAN)、SELinux enforcing模式、Verified Boot防止持久化、定期安全更新、exploit缓解机制（ASLR/DEP/CFI）。
   </details>

6. **权限提升路径**  
   一个普通应用如何合法地获得系统级别的能力？分析至少两种不同的路径及其安全含义。  
   **Hint**: 考虑Android企业功能和辅助功能

   <details>
   <summary>答案</summary>
   
   合法路径：1) DeviceOwner/ProfileOwner通过企业部署获得管理权限 2) AccessibilityService获得UI控制能力 3) DeviceAdminReceiver获得设备管理权限 4) 成为默认应用（如Launcher/SMS）。每种都需要用户明确授权，且有审计机制。
   </details>

7. **SELinux策略设计**  
   为一个需要访问摄像头硬件的自定义HAL服务设计SELinux策略，需要定义哪些规则？  
   **Hint**: 考虑文件访问、binder通信、属性访问

   <details>
   <summary>答案</summary>
   
   需要定义：1) 新的domain类型 2) 文件访问规则(allow domain camera_device:chr_file rw_file_perms) 3) binder通信规则与cameraserver 4) 属性读取权限 5) 设备节点标签 6) 域转换规则。遵循最小权限原则。
   </details>

8. **跨平台安全对比**  
   分析Android的共享UID机制与iOS的App Group在安全性和功能性上的差异，哪种设计更优？  
   **Hint**: 考虑隔离性、灵活性和向后兼容性

   <details>
   <summary>答案</summary>
   
   Android共享UID完全共享进程空间，安全边界模糊但灵活；iOS App Group仅共享特定数据，保持进程隔离。iOS设计更安全但受限，Android在Android 14后限制新应用使用共享UID，推荐使用content provider等IPC机制。
   </details>

## 常见陷阱与错误

1. **权限检查时机错误**
   - 错误：只在UI层检查权限
   - 正确：在实际访问资源前进行权限检查，使用`ContextCompat.checkSelfPermission()`

2. **SELinux策略过度宽松**
   - 错误：使用permissive模式或过宽的allow规则
   - 正确：遵循最小权限原则，使用audit2allow谨慎生成规则

3. **忽视共享存储安全**
   - 错误：在外部存储保存敏感数据
   - 正确：使用应用私有存储或加密敏感数据

4. **不当的UID/PID验证**
   - 错误：仅依赖`Binder.getCallingUid()`而不清空调用身份
   - 正确：使用`Binder.clearCallingIdentity()`确保正确的安全上下文

5. **运行时权限请求不当**
   - 错误：一次请求所有权限或在应用启动时请求
   - 正确：在需要时请求相关权限，提供清晰的使用说明

## 最佳实践检查清单

- [ ] 应用是否遵循最小权限原则，只请求必要的权限？
- [ ] 是否正确处理运行时权限的各种状态（授予、拒绝、不再询问）？
- [ ] 敏感数据是否存储在应用私有目录而非共享存储？
- [ ] 是否为自定义系统组件编写了适当的SELinux策略？
- [ ] IPC接口是否实施了proper的调用者身份验证？
- [ ] 是否使用了Android提供的加密API保护敏感数据？
- [ ] 是否定期更新依赖库以修复已知安全漏洞？
- [ ] 是否在发布前进行了安全测试（如使用StrictMode）？
- [ ] 网络通信是否使用HTTPS并正确验证证书？
- [ ] 是否避免了动态加载代码等危险操作？