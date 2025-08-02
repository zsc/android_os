# 第13章：Android安全模型

Android系统采用多层次、纵深防御的安全架构，从Linux内核到应用层都实施了严格的安全控制。本章将深入剖析Android的安全机制，包括应用沙箱、权限系统、SELinux强制访问控制等核心组件，并与iOS等系统进行技术对比，帮助读者理解移动操作系统安全设计的最佳实践。

## 13.1 应用沙箱机制

### 13.1.1 UID隔离基础

Android继承了Linux的多用户机制，但创新性地将其用于应用隔离。每个应用在安装时都会分配唯一的UID（用户ID），通常从10000开始递增。这种设计使得每个应用都运行在独立的Linux用户空间中。

**UID分配策略**：
- 普通应用：UID范围10000-19999 (FIRST_APPLICATION_UID到LAST_APPLICATION_UID)
- 隔离进程：99000-99999 (FIRST_ISOLATED_UID到LAST_ISOLATED_UID)
- 应用缓存：20000-29999 (FIRST_APP_CACHE_UID到LAST_APP_CACHE_UID)
- 系统UID预定义值：
  - root (0): 超级用户权限
  - system (1000): 系统服务器进程
  - radio (1001): 电话子系统
  - bluetooth (1002): 蓝牙服务
  - graphics (1003): 图形相关服务
  - input (1004): 输入子系统
  - audio (1005): 音频服务
  - camera (1006): 相机服务
  - log (1007): 日志服务
  - compass (1008): 传感器服务
  - mount (1009): 存储挂载服务
  - wifi (1010): WiFi服务
  - adb (1011): ADB调试服务
  - install (1012): 安装服务
  - media (1013): 媒体服务器
  - dhcp (1014): DHCP客户端
  - sdcard_rw (1015): SD卡读写
  - vpn (1016): VPN服务
  - keystore (1017): 密钥存储
  - usb (1018): USB服务
  - drm (1019): DRM服务
  - mdnsr (1020): mDNS服务
  - gps (1021): GPS服务
  - media_rw (1023): 媒体存储读写
  - mtp (1024): MTP服务
  - nfc (1027): NFC服务
  - shell (2000): Shell用户

**UID分配实现**：
- PackageManagerService在应用安装时通过`mSettings.addUserIdLPw()`分配UID
- Settings类维护packages.xml中的UID映射关系
- 对于新安装应用，通过`acquireAndRegisterNewAppIdLPw()`获取未使用的UID
- 系统应用使用AndroidManifest.xml中android:sharedUserId指定预定义UID
- 共享UID机制允许签名相同的应用共享进程和数据，但Android 29+开始不推荐使用

**多用户支持**：
Android支持多用户模式，实际UID计算公式：
```
实际UID = userId * 100000 + appId
```
其中userId是用户ID（主用户为0），appId是应用的基础ID。这使得同一应用在不同用户下有不同的UID。

**UID权限映射**：
每个UID对应特定的Linux权限组（GID），控制对系统资源的访问：
- inet (3003): 网络套接字创建
- sdcard_r (1028): SD卡读取
- sdcard_rw (1015): SD卡读写
- bluetooth (1002): 蓝牙设备访问
- camera (1006): 相机设备访问

### 13.1.2 进程隔离实现

每个应用进程通过Zygote fork创建，继承了完整的安全上下文。Android的进程隔离不仅依赖Linux基础机制，还增加了多层安全增强。

**1. 进程创建与隔离**：

*Zygote fork流程*：
- Zygote预加载常用类和资源，作为应用进程模板
- ActivityManagerService通过`Process.start()`请求创建新进程
- Zygote收到socket请求后，执行`Zygote.forkAndSpecialize()`
- fork后的子进程执行`handleChildProc()`设置安全环境：
  - 设置进程UID/GID：`setuid()`/`setgid()`
  - 设置补充组：`setgroups()`配置权限组
  - 设置能力（capabilities）：`capset()`限制特权
  - 切换SELinux域：`selinux_android_setcontext()`
  - 设置进程优先级和调度策略
  - 配置资源限制（rlimits）

*内存隔离机制*：
- 每个进程独立的虚拟地址空间
- ASLR（地址空间布局随机化）增加攻击难度
- DEP/NX（数据执行保护）防止代码注入
- 进程间不能直接访问对方内存，必须通过Binder IPC

**2. 文件系统隔离**：

*应用私有存储结构*：
```
/data/data/<package_name>/
├── cache/          # 缓存目录，可被系统清理
├── code_cache/     # 运行时代码缓存
├── databases/      # SQLite数据库
├── files/          # 应用私有文件
├── lib/            # native库软链接
├── shared_prefs/   # SharedPreferences
└── no_backup/      # 不参与备份的文件
```

*权限设置*：
- 目录权限：700 (rwx------) 仅应用UID可访问
- 文件默认权限：600 (rw-------) 
- 通过`Context.MODE_PRIVATE`创建的文件仅本应用可访问
- `Context.MODE_WORLD_READABLE/WRITEABLE`已废弃（安全原因）

*外部存储隔离*：
- Android 10前：/sdcard通过FUSE实现，所有应用可见
- Android 10+：Scoped Storage，应用专属目录：
  - `/storage/emulated/0/Android/data/<package_name>/`
  - `/storage/emulated/0/Android/media/<package_name>/`
  - `/storage/emulated/0/Android/obb/<package_name>/`

**3. IPC安全限制**：

*Binder安全检查*：
- 内核驱动级别的UID/PID验证
- 每次Binder调用自动传递调用者身份
- 服务端通过以下API获取调用者信息：
  - `Binder.getCallingUid()`: 获取调用者UID
  - `Binder.getCallingPid()`: 获取调用者PID  
  - `Binder.getCallingUserHandle()`: 获取调用者用户
- 身份切换机制：
  - `Binder.clearCallingIdentity()`: 临时切换到自己的身份
  - `Binder.restoreCallingIdentity()`: 恢复调用者身份

*权限enforcement位置*：
- 系统服务入口：检查调用者是否有required权限
- Context API层：`enforceCallingPermission()`
- Binder接口：通过AIDL注解`@EnforcePermission`
- Native层：通过`IPCThreadState::getCallingUid()`

**4. 其他隔离机制**：

*命名空间隔离*：
- Mount namespace：隔离文件系统挂载点
- PID namespace：进程ID空间隔离（部分使用）
- Network namespace：网络栈隔离（VPN应用）

*资源限制*：
- CPU使用：通过cgroups限制
- 内存限制：通过memory cgroups和oom_adj
- 文件描述符限制：防止资源耗尽攻击

### 13.1.3 存储沙箱演进

Android存储模型经历了重大变革，从早期的开放模型逐步演进到严格的隔离模型。

**存储架构基础**：

*存储类型分类*：
1. **内部存储** (/data分区)
   - 应用私有目录：`/data/data/<package_name>/`
   - 应用APK和库：`/data/app/<package_name>/`
   - 用户数据：`/data/user/<userId>/<package_name>/`
   - 加密特性：默认启用FBE（File-Based Encryption）

2. **外部存储** (/storage分区)
   - 主存储：`/storage/emulated/0/`
   - SD卡：`/storage/<sdcard_id>/`
   - USB存储：`/storage/<usb_id>/`

**Android 10之前的存储模型**：

*FUSE实现机制*：
- sdcard daemon通过FUSE在用户空间实现文件系统
- 基于调用者UID进行权限检查
- 性能开销：每次文件操作需要用户态/内核态切换

*权限控制*：
```
READ_EXTERNAL_STORAGE：
- 读取/sdcard所有文件
- 自动包含媒体文件访问

WRITE_EXTERNAL_STORAGE：
- 写入/sdcard任意位置
- 隐含READ权限
- 可创建任意目录结构
```

*安全问题*：
- 应用可扫描整个外部存储
- 无法阻止恶意应用窃取其他应用数据
- 用户文件（照片、文档）完全暴露

**Android 10 Scoped Storage**：

*核心设计理念*：
- 应用只能访问自己创建的文件
- 共享文件通过系统API访问
- 用户授权的细粒度控制

*实现机制*：
1. **应用专属目录**：
   ```
   Context.getExternalFilesDir() -> /storage/emulated/0/Android/data/<pkg>/files/
   Context.getExternalCacheDir() -> /storage/emulated/0/Android/data/<pkg>/cache/
   Context.getExternalMediaDirs() -> /storage/emulated/0/Android/media/<pkg>/
   ```
   - 无需权限即可访问
   - 应用卸载时自动清理
   - 不会被媒体扫描器索引（除media目录）

2. **MediaStore访问**：
   ```
   媒体集合URI：
   - MediaStore.Images.Media.EXTERNAL_CONTENT_URI
   - MediaStore.Video.Media.EXTERNAL_CONTENT_URI  
   - MediaStore.Audio.Media.EXTERNAL_CONTENT_URI
   - MediaStore.Downloads.EXTERNAL_CONTENT_URI (API 29+)
   ```
   - 通过ContentResolver查询和访问
   - 自动过滤仅显示应用创建的文件
   - 访问其他应用文件需要用户选择

3. **Storage Access Framework (SAF)**：
   - `ACTION_OPEN_DOCUMENT`: 选择单个文件
   - `ACTION_OPEN_DOCUMENT_TREE`: 选择目录树
   - `ACTION_CREATE_DOCUMENT`: 创建新文件
   - 返回持久化的URI权限

*兼容性措施*：
- `requestLegacyExternalStorage="true"`: 临时退出（targetSdk<30）
- `preserveLegacyExternalStorage="true"`: 升级时保留旧行为

**Android 11+增强**：

*新增限制*：
1. **文件路径访问限制**：
   - 不能通过路径访问其他应用的文件
   - Environment.getExternalStorageDirectory()废弃
   - File API仅限应用专属目录

2. **MANAGE_EXTERNAL_STORAGE权限**：
   - 特殊权限，需要用户在设置中授予
   - 仅限文件管理器等特殊用例
   - Google Play审核严格限制
   - 通过`ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION`请求

3. **媒体文件访问优化**：
   - 批量媒体文件操作：`MediaStore.createWriteRequest()`
   - 原生文件路径访问（有限场景）
   - IS_PENDING标志：处理大文件时防止其他应用访问

**Android 12+进一步改进**：

*应用存储访问指示器*：
- 状态栏显示存储访问图标
- 隐私仪表板记录访问历史

*媒体权限细分*：
```
READ_MEDIA_IMAGES: 仅图片访问
READ_MEDIA_VIDEO: 仅视频访问  
READ_MEDIA_AUDIO: 仅音频访问
替代原有的READ_EXTERNAL_STORAGE
```

**Android 13+最新变化**：

*照片选择器*：
- 系统级UI选择特定照片/视频
- 无需完整媒体权限
- `ACTION_PICK_IMAGES`意图

*细粒度媒体权限*：
- 针对音频、图片、视频的独立权限
- 更精确的用户控制

### 13.1.4 网络沙箱

Android实现了细粒度的网络访问控制，从应用层到内核层都有相应的安全机制。

**1. 网络权限控制**：

*INTERNET权限机制*：
- 权限检查位置：socket()系统调用
- 内核通过GID检查：inet组(GID=3003)
- 无INTERNET权限的应用无法创建网络socket
- 实现代码路径：`af_inet.c`中的权限检查

*其他网络相关权限*：
```
ACCESS_NETWORK_STATE: 读取网络连接状态
ACCESS_WIFI_STATE: 读取WiFi状态信息
CHANGE_NETWORK_STATE: 修改网络连接
CHANGE_WIFI_STATE: 修改WiFi状态
ACCESS_FINE_LOCATION: WiFi/基站定位需要
BIND_VPN_SERVICE: 创建VPN服务
```

**2. 网络流量控制（Netd守护进程）**：

*UID级别的iptables规则*：
```bash
# 按UID允许/拒绝网络访问
iptables -A fw_OUTPUT -m owner --uid-owner <uid> -j ACCEPT/DROP

# 按UID路由到特定网络接口
ip rule add uidrange <start>-<end> table <table_id>

# 带宽限制
iptables -A bw_costly_<iface> -m owner --uid-owner <uid> -j bw_penalty_box
```

*网络策略实施*：
- NetworkPolicyManagerService管理网络策略
- 通过Netd设置iptables规则
- 支持按UID的流量统计和限制
- 后台数据限制实现

**3. VPN隔离机制**：

*Per-app VPN*：
- VpnService.Builder.addAllowedApplication()：指定应用使用VPN
- VpnService.Builder.addDisallowedApplication()：排除特定应用
- 通过路由表和iptables规则实现隔离

*实现原理*：
```
1. 创建tun接口：/dev/tun
2. 设置路由规则：
   ip rule add uidrange <uid> table <vpn_table>
   ip route add default dev tun0 table <vpn_table>
3. iptables标记：
   iptables -t mangle -A OUTPUT -m owner --uid-owner <uid> -j MARK --set-mark <vpn_mark>
```

*Always-on VPN*：
- 系统启动时自动建立
- 可配置阻止非VPN流量
- DevicePolicyManager控制企业VPN

**4. 网络安全增强**：

*DNS-over-TLS (Android 9+)*：
- Private DNS设置
- 自动检测DNS服务器TLS支持
- 防止DNS劫持和监听

*网络安全配置 (Android 7+)*：
```xml
<!-- network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </base-config>
    <domain-config>
        <domain includeSubdomains="true">example.com</domain>
        <pin-set expiration="2025-01-01">
            <pin digest="SHA-256">base64==</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

*证书固定（Certificate Pinning）*：
- 应用级：NetworkSecurityConfig
- 代码级：自定义TrustManager
- 系统级：系统证书存储管理

**5. 网络隔离高级特性**：

*多网络API (Android 5+)*：
- ConnectivityManager.bindProcessToNetwork()：进程绑定到特定网络
- Network.openConnection()：通过特定网络建立连接
- 支持同时连接多个网络（WiFi+蜂窝）

*网络切片（Network Slicing）*：
- 5G网络支持
- 不同应用使用不同网络切片
- QoS保证和隔离

*企业网络隔离*：
- Work Profile网络隔离
- 企业VPN与个人VPN分离
- MDM控制的网络策略

**6. 网络审计与监控**：

*流量统计*：
- NetworkStatsService收集UID级别统计
- /proc/net/xt_qtaguid/stats接口
- TrafficStats API供应用查询

*网络日志*：
- tcpdump需要root权限
- 应用级：OkHttp拦截器等
- 系统级：netlog调试

*防火墙日志*：
```bash
# 启用iptables日志
iptables -A INPUT -j LOG --log-prefix "[FW_INPUT] "
# 查看日志
logcat -s netd
```

## 13.2 权限系统演进

### 13.2.1 权限模型基础

Android权限系统基于最小权限原则，通过细粒度的权限控制保护用户隐私和系统安全。

**权限层级体系**：

*基本权限级别*：
1. **normal** (普通权限)
   - 低风险，安装时自动授予
   - 例如：INTERNET, ACCESS_NETWORK_STATE, VIBRATE
   - 不涉及用户隐私数据或其他应用
   - 占据大部分系统权限

2. **dangerous** (危险权限)
   - 涉及用户隐私或敏感数据
   - Android 6.0+需要运行时请求
   - 按权限组管理，授予一个则整组授予
   - 典型权限组：
     - CALENDAR: READ_CALENDAR, WRITE_CALENDAR
     - CAMERA: CAMERA
     - CONTACTS: READ_CONTACTS, WRITE_CONTACTS, GET_ACCOUNTS
     - LOCATION: ACCESS_FINE_LOCATION, ACCESS_COARSE_LOCATION, ACCESS_BACKGROUND_LOCATION
     - MICROPHONE: RECORD_AUDIO
     - PHONE: READ_PHONE_STATE, CALL_PHONE, READ_CALL_LOG, etc.
     - SENSORS: BODY_SENSORS
     - SMS: SEND_SMS, RECEIVE_SMS, READ_SMS, etc.
     - STORAGE: READ_EXTERNAL_STORAGE, WRITE_EXTERNAL_STORAGE

3. **signature** (签名权限)
   - 仅授予与声明者相同签名的应用
   - 用于同一开发者的应用间共享功能
   - 例如：SIGNAL_PERSISTENT_PROCESSES

4. **signature|privileged** (旧称signatureOrSystem)
   - 签名相同或预装在/system/priv-app
   - 系统核心功能使用
   - 例如：INSTALL_PACKAGES, DELETE_PACKAGES

5. **privileged** (特权权限)
   - 仅预装在/system/priv-app的应用
   - 需要在privapp-permissions.xml中白名单
   - 例如：CHANGE_COMPONENT_ENABLED_STATE

6. **development** (开发权限)
   - 仅在开发者选项启用时可用
   - 例如：SET_DEBUG_APP, DUMP

*权限保护级别属性*：
```xml
<!-- 在framework-res/AndroidManifest.xml中定义 -->
<permission android:name="android.permission.CAMERA"
    android:permissionGroup="android.permission-group.CAMERA"
    android:protectionLevel="dangerous"
    android:label="@string/permlab_camera"
    android:description="@string/permdesc_camera" />
```

**权限实现架构**：

*权限存储与管理*：
1. **静态定义**：
   - 系统权限：framework-res/AndroidManifest.xml
   - OEM权限：/system/etc/permissions/
   - 应用自定义权限：应用AndroidManifest.xml

2. **运行时存储**：
   - `/data/system/packages.xml`：应用权限授予状态
   - `/data/misc_de/0/apexdata/com.android.permission/`：运行时权限状态
   - 内存中：PackageManagerService.mSettings

3. **权限授予记录**：
   ```xml
   <!-- packages.xml 示例 -->
   <package name="com.example.app" >
       <perms>
           <item name="android.permission.CAMERA" granted="true" flags="0" />
           <item name="android.permission.INTERNET" granted="true" flags="0" />
       </perms>
   </package>
   ```

**权限检查机制**：

*检查流程*：
1. **应用层检查**：
   ```
   Context.checkSelfPermission(permission)
   → ContextImpl.checkPermission(permission, pid, uid)
   → ActivityManager.checkPermission(permission, pid, uid)
   ```

2. **系统服务检查**：
   ```
   ActivityManagerService.checkPermission()
   → PackageManager.checkUidPermission(permission, uid)
   → PermissionManagerService.checkUidPermission()
   ```

3. **强制执行**：
   ```
   Context.enforcePermission(permission, pid, uid, message)
   → 抛出SecurityException如果无权限
   ```

*性能优化*：
- PermissionCache缓存常用权限检查结果
- 批量权限检查API减少IPC次数

**权限代理机制**：

*AppOps系统*：
- 精细控制应用操作
- 跟踪权限使用情况
- 支持临时禁用权限

```
// AppOps操作示例
AppOpsManager.noteOp(AppOpsManager.OP_CAMERA, uid, packageName)
→ 记录权限使用
→ 可能返回AppOpsManager.MODE_IGNORED禁用操作
```

*权限委托*：
- URI权限：`Intent.FLAG_GRANT_READ_URI_PERMISSION`
- 临时权限授予给其他应用
- 用于分享文件等场景

### 13.2.2 运行时权限（Android 6.0+）

运行时权限彻底改变了危险权限的授予方式，大幅提升了用户对隐私的控制能力。

**核心设计理念**：

1. **上下文相关授权**：用户在使用功能时授权，更容易理解
2. **可撤销权限**：用户可随时在设置中撤销已授予权限
3. **权限组简化**：减少用户决策次数
4. **兼容性设计**：旧应用保持安装时授权行为

**实现架构**：

*系统组件*：
1. **PermissionController** (原名PackageInstaller)
   - 系统应用，负责权限UI展示
   - 位置：/system/priv-app/PermissionController/
   - 提供GrantPermissionsActivity处理权限请求
   - 管理权限设置页面

2. **PermissionManagerService**
   - 系统服务，管理权限状态
   - 处理权限授予/撤销逻辑
   - 维护权限组关系
   - 同步权限状态到存储

3. **AppOpsService**
   - 跟踪权限使用情况
   - 支持临时禁用权限
   - 提供使用统计数据

**权限请求流程**：

```
1. 应用调用：requestPermissions(permissions[], requestCode)
   ↓
2. ActivityManagerService处理请求
   ↓
3. 启动PermissionController中的GrantPermissionsActivity
   ↓
4. 用户做出选择（允许/拒绝/仅在使用期间允许）
   ↓
5. PermissionManagerService更新权限状态
   ↓
6. 回调应用的onRequestPermissionsResult()
```

**权限状态管理**：

*权限授予状态*：
- PERMISSION_GRANTED：已授予
- PERMISSION_DENIED：未授予
- PERMISSION_DENIED_APP_OP：通过AppOps禁用

*权限标志*：
```java
// PermissionInfo.java中的标志
FLAG_COSTS_MONEY = 1<<0;        // 可能产生费用
FLAG_REMOVED = 1<<1;             // 已移除的权限
FLAG_HARD_RESTRICTED = 1<<2;     // 硬性限制权限
FLAG_SOFT_RESTRICTED = 1<<3;     // 软性限制权限
FLAG_IMMUTABLY_RESTRICTED = 1<<4; // 不可变限制
```

*持久化存储*：
```xml
<!-- runtime-permissions.xml -->
<runtime-permissions fingerprint="...">
  <pkg name="com.example.app">
    <perm name="android.permission.CAMERA" granted="true" />
    <perm name="android.permission.LOCATION" granted="false" />
  </pkg>
</runtime-permissions>
```

**权限组机制**：

*权限组定义*：
```xml
<permission-group android:name="android.permission-group.LOCATION"
    android:icon="@drawable/perm_group_location"
    android:label="@string/permgroup_location"
    android:description="@string/permgroup_location_description"
    android:priority="400" />
```

*权限组授予规则*：
- 同组权限一起授予（Android 6.0-7.1）
- Android 8.0+改为单独授予，但UI仍按组展示
- 组内已授予权限不再弹窗

**高级特性**：

*自动撤销机制 (Android 11+)*：
```kotlin
// 应用长时间未使用时自动撤销权限
val unusedAppRestrictionsStatus = 
    packageManager.getUnusedAppRestrictionsStatus()
    
when (unusedAppRestrictionsStatus) {
    FEATURE_NOT_AVAILABLE -> // 功能不可用
    DISABLED -> // 已禁用自动撤销
    API_30_BACKPORT, API_31 -> // 启用中
}
```

*单次授权 (Android 11+)*：
- “仅此一次”选项
- 应用退到后台后自动撤销
- 通过ActivityManager监听应用生命周期

*权限使用说明*：
```java
// 判断是否需要显示使用说明
if (shouldShowRequestPermissionRationale(permission)) {
    // 显示为什么需要这个权限
    // 然后再次请求
}
```

**兼容性处理**：

*targetSdkVersion影响*：
- < 23：保持安装时授权，但用户可在设置中撤销
- >= 23：必须实现运行时权限逻辑

*降级处理*：
```java
// 旧应用在权限被撤销后
// 相关API返回空值或默认值
// 例如：getLastKnownLocation()返回null
```

### 13.2.3 权限使用审计（Android 10+）

Android引入了更透明的权限使用跟踪，让用户能够清楚了解应用如何使用授予的权限。

**1. 位置权限改革**：

*三级位置权限体系*：

1. **前台位置权限**
   - ACCESS_COARSE_LOCATION：粗略位置（精度约2km）
   - ACCESS_FINE_LOCATION：精确位置（GPS级别）
   - 仅在应用前台运行时可用
   - 包括前台服务场景

2. **后台位置权限** (Android 10+)
   - ACCESS_BACKGROUND_LOCATION
   - 必须先获得前台位置权限
   - 需要单独请求，不能与前台一起
   - 用户选择“始终允许”才授予

3. **用户选项增强** (Android 11+)
   - “仅在使用应用时”
   - “仅此一次”
   - “拒绝”
   - “拒绝且不再询问”

*位置精度降级*：
```java
// 用户可以选择仅授予粗略位置
// 即使应用请求ACCESS_FINE_LOCATION
LocationManager locationManager = getSystemService(LocationManager.class);
if (locationManager.isProviderEnabled(LocationManager.GPS_PROVIDER)) {
    // 可能被系统降级为网络位置
}
```

**2. 权限使用指示器**：

*实时指示器 (Android 12+)*：
- 状态栏右上角绿色圆点
- 点击显示使用详情
- 摄像头：📷 图标
- 麦克风：🎤 图标
- 位置：📍 图标

*指示器实现*：
```java
// PermissionUsageHelper.java
public class PermissionUsageHelper {
    // 监听权限使用
    private void startListeningForPermissionUsage() {
        mAppOpsManager.startWatchingActive(
            OPSTR_CAMERA, OPSTR_RECORD_AUDIO,
            mExecutor, mActiveOpListener);
    }
    
    // 更新指示器状态
    private void updateIndicatorState(String op, boolean active) {
        if (active) {
            showIndicator(op);
        } else {
            hideIndicator(op);
        }
    }
}
```

**3. 隐私仪表板**：

*功能特性*：
- 24小时内权限使用时间轴
- 按应用分组显示
- 支持撤销权限快捷入口
- 显示访问次数和时长

*数据收集*：
```java
// AppOpsService记录使用事件
public class AppOpsService {
    private void noteOperationUnchecked(int code, int uid, 
            String packageName, String attributionTag) {
        // 记录操作时间戳
        Op op = getOpLocked(ops, code, uid, true);
        op.noteOpCount++;
        op.time[uidState.state] = System.currentTimeMillis();
        
        // 发送到PermissionController
        mHistoricalRegistry.noteOp(code, uid, packageName);
    }
}
```

*UI展示*：
- 设置 → 隐私 → 权限管理器
- 点击具体权限查看使用历史
- 支持按时间筛选

**4. 权限使用审计API**：

*开发者API*：
```java
// 获取权限使用记录
AppOpsManager appOpsManager = getSystemService(AppOpsManager.class);
List<AppOpsManager.PackageOps> packages = 
    appOpsManager.getPackagesForOps(new int[]{OP_CAMERA});

for (PackageOps pkg : packages) {
    for (OpEntry op : pkg.getOps()) {
        long lastAccessTime = op.getLastAccessTime();
        int accessCount = op.getAccessCount();
    }
}
```

*系统审计日志*：
```bash
# 查看权限使用日志
adb shell dumpsys appops

# 输出示例
Uid 10123:
  Package com.example.app:
    CAMERA (allow / switch COARSE_LOCATION=allow):
      Access: [fg-s] 2024-01-15 10:30:15.123 (-5m32s)
      Reject: [bg] 2024-01-15 09:15:00.000 (-1h20m47s)
```

**5. 数据访问审计 (Android 11+)**：

*数据访问记录*：
```java
// 应用可以记录自己的数据访问
AppOpsManager.OnOpNotedCallback callback = new OnOpNotedCallback() {
    @Override
    public void onOpNoted(String op, String attributionTag) {
        Log.d(TAG, "Permission " + op + " was used");
    }
};
appOpsManager.setOnOpNotedCallback(executor, callback);
```

*归因标签（Attribution Tags）*：
```xml
<!-- AndroidManifest.xml -->
<attribution
    android:tag="location_feature"
    android:label="@string/location_feature_label" />
```

```java
// 使用归因标签
Context attributionContext = 
    createAttributionContext("location_feature");
LocationManager locationManager = 
    attributionContext.getSystemService(LocationManager.class);
// 此时位置访问会记录归因标签
```

**6. 自动重置机制**：

*未使用应用权限自动重置 (Android 11+)*：
- 几个月未使用的应用
- 自动撤销运行时权限
- 用户可以禁用此功能

```java
// 检查自动重置状态
PackageManager pm = getPackageManager();
if (pm.isAutoRevokeWhitelisted()) {
    // 应用已豁免自动重置
}

// 引导用户禁用自动重置
Intent intent = new Intent(
    Intent.ACTION_AUTO_REVOKE_PERMISSIONS,
    Uri.parse("package:" + getPackageName()));
startActivity(intent);
```

### 13.2.4 特殊权限机制

某些敏感操作需要特殊权限，这些权限不能通过普通的运行时权限流程授予。

**1. 系统设置权限**：

*SYSTEM_ALERT_WINDOW (悬浮窗)*：
```java
// 检查权限
if (!Settings.canDrawOverlays(this)) {
    // 请求权限
    Intent intent = new Intent(
        Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
        Uri.parse("package:" + getPackageName()));
    startActivityForResult(intent, REQUEST_CODE);
}

// 使用悬浮窗
WindowManager.LayoutParams params = new WindowManager.LayoutParams(
    WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
    WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
    PixelFormat.TRANSLUCENT);
windowManager.addView(floatingView, params);
```

*WRITE_SETTINGS (修改系统设置)*：
```java
// 检查权限
if (!Settings.System.canWrite(this)) {
    Intent intent = new Intent(
        Settings.ACTION_MANAGE_WRITE_SETTINGS,
        Uri.parse("package:" + getPackageName()));
    startActivityForResult(intent, REQUEST_CODE);
}

// 修改设置
Settings.System.putInt(
    getContentResolver(),
    Settings.System.SCREEN_BRIGHTNESS, 
    brightness);
```

*REQUEST_INSTALL_PACKAGES (安装应用)*：
```java
// Android 8.0+需要此权限
if (!getPackageManager().canRequestPackageInstalls()) {
    Intent intent = new Intent(
        Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES,
        Uri.parse("package:" + getPackageName()));
    startActivityForResult(intent, REQUEST_CODE);
}

// 安装APK
Intent installIntent = new Intent(Intent.ACTION_VIEW);
installIntent.setDataAndType(apkUri, 
    "application/vnd.android.package-archive");
installIntent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
startActivity(installIntent);
```

**2. 辅助功能权限**：

*AccessibilityService*：
```xml
<!-- 服务声明 -->
<service android:name=".MyAccessibilityService"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService" />
    </intent-filter>
    <meta-data
        android:name="android.accessibilityservice"
        android:resource="@xml/accessibility_service_config" />
</service>
```

```xml
<!-- accessibility_service_config.xml -->
<accessibility-service
    android:accessibilityEventTypes="typeAllMask"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:accessibilityFlags="flagReportViewIds|flagRetrieveInteractiveWindows"
    android:canRetrieveWindowContent="true"
    android:canPerformGestures="true"
    android:description="@string/accessibility_service_description" />
```

*能力范围*：
- 读取屏幕内容
- 模拟用户输入
- 监听系统事件
- 执行全局手势

**3. 设备管理权限**：

*DeviceAdminReceiver*：
```xml
<!-- 设备管理器声明 -->
<receiver android:name=".DeviceAdminReceiver"
    android:permission="android.permission.BIND_DEVICE_ADMIN">
    <meta-data android:name="android.app.device_admin"
        android:resource="@xml/device_admin" />
    <intent-filter>
        <action android:name="android.app.action.DEVICE_ADMIN_ENABLED" />
    </intent-filter>
</receiver>
```

```java
// 激活设备管理器
ComponentName adminComponent = new ComponentName(this, DeviceAdminReceiver.class);
Intent intent = new Intent(DevicePolicyManager.ACTION_ADD_DEVICE_ADMIN);
intent.putExtra(DevicePolicyManager.EXTRA_DEVICE_ADMIN, adminComponent);
startActivityForResult(intent, REQUEST_ENABLE);

// 使用管理功能
DevicePolicyManager dpm = getSystemService(DevicePolicyManager.class);
dpm.lockNow(); // 锁定设备
dpm.resetPassword("newpassword", 0); // 重置密码
dpm.wipeData(0); // 清除数据
```

*DeviceOwner/ProfileOwner*：
```bash
# 设置DeviceOwner（需要设备未配置）
adb shell dpm set-device-owner com.example.app/.DeviceAdminReceiver

# 设置ProfileOwner（Work Profile）
adb shell dpm set-profile-owner com.example.app/.DeviceAdminReceiver --user 10
```

*企业管理能力*：
```java
// DeviceOwner可以：
// 1. 卸载任意应用
dpm.setUninstallBlocked(admin, packageName, true);

// 2. 隐藏应用
dpm.setApplicationHidden(admin, packageName, true);

// 3. 设置系统更新策略
dpm.setSystemUpdatePolicy(admin, SystemUpdatePolicy.createAutomaticInstallPolicy());

// 4. 设置全局HTTP代理
dpm.setGlobalProxy(admin, proxyInfo);

// 5. 禁用相机
dpm.setCameraDisabled(admin, true);
```

**4. 通知监听权限**：

*NotificationListenerService*：
```xml
<service android:name=".NotificationListener"
    android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
    <intent-filter>
        <action android:name="android.service.notification.NotificationListenerService" />
    </intent-filter>
</service>
```

```java
public class NotificationListener extends NotificationListenerService {
    @Override
    public void onNotificationPosted(StatusBarNotification sbn) {
        // 获取通知内容
        String packageName = sbn.getPackageName();
        CharSequence title = sbn.getNotification().extras
            .getCharSequence(Notification.EXTRA_TITLE);
        CharSequence text = sbn.getNotification().extras
            .getCharSequence(Notification.EXTRA_TEXT);
    }
    
    @Override
    public void onNotificationRemoved(StatusBarNotification sbn) {
        // 通知被移除
    }
}
```

**5. VPN服务权限**：

*VpnService*：
```java
public class MyVpnService extends VpnService {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // 配置VPN
        Builder builder = new Builder();
        builder.setSession("MyVPN")
            .addAddress("10.0.0.2", 24)
            .addDnsServer("8.8.8.8")
            .addRoute("0.0.0.0", 0)
            .setMtu(1500);
            
        // 允许/禁止特定应用
        builder.addAllowedApplication("com.example.app");
        builder.addDisallowedApplication("com.example.blocked");
        
        ParcelFileDescriptor interface = builder.establish();
        // 处理VPN流量
        return START_STICKY;
    }
}
```

**6. 特殊权限管理**：

*权限查询API*：
```java
// 获取所有特殊权限应用
PackageManager pm = getPackageManager();
List<PackageInfo> packages = pm.getPackagesHoldingPermissions(
    new String[]{Manifest.permission.SYSTEM_ALERT_WINDOW},
    PackageManager.GET_PERMISSIONS);

// 检查多个特殊权限
AppOpsManager appOps = getSystemService(AppOpsManager.class);
int mode = appOps.checkOpNoThrow(
    AppOpsManager.OPSTR_SYSTEM_ALERT_WINDOW,
    uid, packageName);
boolean granted = mode == AppOpsManager.MODE_ALLOWED;
```

*用户体验优化*：
```java
// 提示用户为什么需要特殊权限
private void requestSpecialPermission() {
    new AlertDialog.Builder(this)
        .setTitle("需要悬浮窗权限")
        .setMessage("为了显示悬浮计时器，需要您授予悬浮窗权限")
        .setPositiveButton("去设置", (dialog, which) -> {
            Intent intent = new Intent(
                Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                Uri.parse("package:" + getPackageName()));
            startActivityForResult(intent, REQUEST_CODE);
        })
        .setNegativeButton("取消", null)
        .show();
}
```

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