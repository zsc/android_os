# 第14章：密钥管理与硬件安全

在现代移动设备中，密钥管理和硬件安全构成了整个安全体系的基石。Android从早期版本开始就不断强化其密钥管理机制，从纯软件实现逐步演进到依赖专用硬件安全模块。本章将深入剖析Android Keystore系统、Trusty TEE（可信执行环境）、硬件密钥认证以及安全启动链的实现原理。我们将重点关注这些机制如何协同工作以保护用户数据和系统完整性，并与iOS的安全架构进行对比分析。通过本章学习，读者将掌握Android硬件安全的核心概念和实现细节，理解如何在应用开发中正确使用这些安全特性。

## Keystore系统架构

### Keystore演进历程

Android Keystore系统经历了多个重要演进阶段：

1. **Android 4.3 (API 18)**：引入基础Keystore API，仅支持软件密钥存储
2. **Android 6.0 (API 23)**：强制硬件支持，引入密钥使用授权机制
3. **Android 9.0 (API 28)**：StrongBox Keymaster引入，支持独立安全芯片
4. **Android 11 (API 30)**：Keymaster 4.1，增强密钥认证功能

### Keystore架构分层

Keystore系统采用分层架构设计，从上到下包括：

**应用层接口**
- KeyStore类：Java层API入口，提供密钥的生成、导入、使用接口
- KeyGenerator/KeyPairGenerator：密钥生成器，支持对称和非对称密钥
- Cipher/Signature：加密和签名操作接口

**系统服务层**
- keystore守护进程：运行在独立进程中，管理密钥存储和访问控制
- KeystoreService：Binder服务接口，处理应用请求
- 密钥数据库：使用加密的SQLite存储密钥元数据

**HAL层**
- Keymaster HAL：硬件抽象层接口，定义标准密钥操作
- IKeymasterDevice：HIDL/AIDL接口定义
- 厂商实现：对接具体硬件安全模块

**硬件层**
- TEE实现：在可信执行环境中运行的Keymaster TA
- 独立安全芯片：如Titan M等专用安全处理器
- SoC集成方案：集成在主SoC中的安全子系统

### 密钥存储机制

Android支持多种密钥存储后端：

**软件密钥存储**
- 使用keystore守护进程的主密钥加密
- 存储在/data/misc/keystore目录
- 主密钥派生自用户凭据（PIN/密码/图案）

**硬件密钥存储**
- 密钥材料永不离开安全硬件
- 仅返回密钥句柄（key blob）给系统
- 支持密钥使用限制（如需要用户认证）

**密钥Blob格式**
```
KeyBlob {
    version: u8,
    key_material: encrypted_bytes,
    auth_tags: AuthorizationSet,
    hw_enforced: AuthorizationSet,
    sw_enforced: AuthorizationSet,
}
```

### 密钥生命周期管理

**密钥生成流程**
1. 应用调用KeyGenerator.generateKey()
2. KeyStore服务验证参数并生成请求
3. 通过HAL调用硬件Keymaster
4. 在TEE/安全芯片中生成密钥
5. 返回加密的key blob存储

**密钥使用授权**
- 用户认证绑定：requireUserAuthentication()
- 时间限制：setUserAuthenticationValidityDurationSeconds()
- 生物识别绑定：setUserAuthenticationRequired()
- 安全锁屏绑定：setUnlockedDeviceRequired()

**密钥销毁机制**
- 显式删除：KeyStore.deleteEntry()
- 应用卸载自动清理
- 用户数据擦除
- 硬件密钥的安全擦除保证

### 与iOS Keychain对比

| 特性 | Android Keystore | iOS Keychain |
|------|-----------------|--------------|
| API设计 | Java密码学架构 | C/Objective-C API |
| 硬件支持 | Keymaster HAL标准化 | Secure Enclave专有 |
| 密钥类型 | 完整密码学算法支持 | 主要支持椭圆曲线 |
| 访问控制 | 基于UID和别名 | 基于entitlement |
| 云同步 | 不支持 | iCloud Keychain |
| 密钥认证 | Key Attestation | DeviceCheck |

### Keystore安全特性

**密钥隔离**
- 进程级隔离：每个应用只能访问自己的密钥
- 用户级隔离：多用户环境下密钥完全隔离
- 密钥别名命名空间：防止密钥名称冲突

**防回滚保护**
- 密钥版本控制
- 单调计数器防止降级攻击
- 与Verified Boot集成

**侧信道防护**
- 时间恒定的密码学实现
- 功耗分析防护
- 电磁泄露防护（硬件相关）

## Trusty TEE架构

### TEE基础概念

可信执行环境（Trusted Execution Environment）是与主操作系统隔离的安全执行环境：

**硬件隔离机制**
- ARM TrustZone：将处理器分为安全世界和普通世界
- 内存隔离：TZASC（TrustZone Address Space Controller）
- 中断隔离：安全中断和非安全中断分离
- 外设隔离：安全外设只能被安全世界访问

**TEE与REE通信**
- SMC（Secure Monitor Call）指令触发世界切换
- 共享内存用于数据传递
- RPC（Remote Procedure Call）机制

### Trusty OS设计

Google开发的Trusty是Android推荐的TEE操作系统：

**Trusty内核特性**
- 微内核架构：最小化TCB（Trusted Computing Base）
- 用户空间隔离：每个TA运行在独立地址空间
- IPC机制：基于端口的进程间通信
- 调度器：支持多任务和优先级调度

**Trusty应用框架**
- libtrusty：核心运行时库
- 存储服务：安全文件系统
- 密码学服务：硬件加速的密码学操作
- IPC服务：与普通世界通信

### Keymaster TA实现

Keymaster是Trusty中最重要的可信应用：

**Keymaster TA架构**
```
Keymaster TA
├── 密钥生成模块
│   ├── RSA密钥生成
│   ├── ECC密钥生成
│   └── AES密钥生成
├── 密钥存储模块
│   ├── 密钥加密/解密
│   ├── 密钥导入/导出
│   └── 密钥元数据管理
├── 密码学操作模块
│   ├── 加密/解密
│   ├── 签名/验证
│   └── 密钥协商
└── 认证模块
    ├── 密钥认证生成
    └── 使用授权验证
```

**安全密钥存储**
- RPMB（Replay Protected Memory Block）：防重放的安全存储
- 密钥加密密钥（KEK）：由硬件唯一密钥派生
- 密钥分层：主密钥→密钥加密密钥→实际密钥

### Gatekeeper TA

Gatekeeper负责用户凭据验证：

**功能设计**
- 密码验证：安全的密码哈希和验证
- 限速机制：防止暴力破解
- 密码历史：防止重复使用旧密码
- 与Keymaster集成：解锁用户绑定的密钥

**防暴力破解**
- 指数退避算法
- 硬件支持的失败计数器
- 30秒超时惩罚机制

### 其他TEE方案对比

| TEE方案 | 特点 | 应用场景 |
|---------|------|----------|
| Trusty | Google开发，开源 | Pixel设备 |
| QSEE | 高通方案，闭源 | 骁龙平台 |
| TrustedCore | 华为方案 | 麒麟平台 |
| Kinibi | 三星方案 | Exynos平台 |
| OP-TEE | 开源参考实现 | 开发测试 |

## 硬件密钥认证

### Key Attestation概述

密钥认证（Key Attestation）是Android提供的一种机制，用于验证密钥确实存储在硬件安全模块中，并且具有特定的属性。这是建立设备信任链的关键技术。

**认证目标**
- 证明密钥在硬件中生成
- 验证密钥属性（算法、大小、用途）
- 确认设备安全状态
- 建立设备身份标识

### 认证证书链结构

Key Attestation生成一个X.509证书链：

**证书链组成**
1. **密钥证书**：包含被认证的公钥和认证扩展
2. **中间证书**：设备特定的认证密钥证书
3. **根证书**：Google硬件认证根证书

**认证扩展结构**
```
KeyDescription ::= SEQUENCE {
    attestationVersion INTEGER,
    attestationSecurityLevel SecurityLevel,
    keymasterVersion INTEGER,
    keymasterSecurityLevel SecurityLevel,
    attestationChallenge OCTET STRING,
    uniqueId OCTET STRING,
    softwareEnforced AuthorizationList,
    teeEnforced AuthorizationList,
}

AuthorizationList ::= SEQUENCE {
    purpose [1] EXPLICIT SET OF INTEGER OPTIONAL,
    algorithm [2] EXPLICIT INTEGER OPTIONAL,
    keySize [3] EXPLICIT INTEGER OPTIONAL,
    digest [5] EXPLICIT SET OF INTEGER OPTIONAL,
    padding [6] EXPLICIT SET OF INTEGER OPTIONAL,
    ecCurve [10] EXPLICIT INTEGER OPTIONAL,
    rsaPublicExponent [200] EXPLICIT INTEGER OPTIONAL,
    rollbackResistance [303] EXPLICIT NULL OPTIONAL,
    rootOfTrust [704] EXPLICIT RootOfTrust OPTIONAL,
    ...
}
```

### 认证流程实现

**客户端认证请求**
1. 生成随机挑战值（challenge）
2. 创建密钥并请求认证
3. 获取认证证书链
4. 发送证书链到服务器

**服务器验证流程**
1. 验证证书链签名
2. 检查根证书可信性
3. 解析认证扩展
4. 验证挑战值匹配
5. 检查设备安全属性

**关键API调用**
- setAttestationChallenge()：设置挑战值
- generateKeyPair()：生成密钥对并认证
- getCertificateChain()：获取认证证书链

### Root of Trust验证

Root of Trust包含设备启动状态信息：

**RootOfTrust结构**
```
RootOfTrust ::= SEQUENCE {
    verifiedBootKey OCTET STRING,
    deviceLocked BOOLEAN,
    verifiedBootState VerifiedBootState,
    verifiedBootHash OCTET STRING,
}

VerifiedBootState ::= ENUMERATED {
    Verified(0),
    SelfSigned(1),
    Unverified(2),
    Failed(3),
}
```

**安全状态判断**
- Verified：设备使用官方签名的系统镜像
- SelfSigned：使用用户自定义密钥签名
- Unverified：未启用verified boot
- Failed：系统完整性验证失败

### SafetyNet/Play Integrity API

除了Key Attestation，Android还提供应用级的设备认证：

**SafetyNet Attestation（已弃用）**
- JWS格式的认证结果
- 包含设备完整性信息
- CTS兼容性检查
- 基本完整性vs设备完整性

**Play Integrity API（推荐）**
- 更强的防篡改保护
- 应用完整性验证
- 账号真实性检查
- 设备活动风险评估

**集成要点**
```
判定标准层次：
1. 设备完整性：未root、官方系统
2. 基本完整性：可能已root但未主动攻击
3. 应用完整性：APK未被篡改
4. 账号可信度：非机器人账号
```

### StrongBox Keymaster

Android 9引入StrongBox，提供更高安全级别：

**StrongBox要求**
- 独立的防篡改硬件
- 独立的CPU、内存、存储
- 真随机数生成器
- 侧信道攻击防护

**与TEE Keymaster对比**
| 特性 | TEE Keymaster | StrongBox |
|------|---------------|-----------|
| 隔离级别 | 软件隔离 | 硬件隔离 |
| 性能 | 较高 | 较低 |
| 抗物理攻击 | 有限 | 强 |
| 成本 | 低 | 高 |
| 支持算法 | 完整 | 有限 |

### 与iOS对比

**iOS设备认证机制**
- DeviceCheck：简单的设备标识
- App Attest：应用完整性认证
- Secure Enclave认证：类似Key Attestation

**主要差异**
1. iOS认证更加封闭和简化
2. Android提供更详细的设备状态
3. iOS强制所有设备支持
4. Android允许更灵活的实现

## 安全启动链

### Verified Boot概述

Android Verified Boot (AVB) 确保设备运行可信的软件：

**启动链信任传递**
```
ROM Boot → Bootloader → Kernel → System
    ↓           ↓          ↓        ↓
  验证下级    验证下级   验证下级  验证应用
```

**保护目标**
- 防止持久性恶意软件
- 检测系统篡改
- 提供回滚保护
- 支持安全更新

### 启动流程详解

**ROM Boot阶段**
- 不可修改的片上ROM代码
- 验证并加载主bootloader
- 初始化硬件安全特性
- 建立初始信任根

**Bootloader验证**
- 使用OEM密钥验证签名
- 检查防回滚版本
- 验证设备锁定状态
- 加载并验证kernel

**Kernel验证**
- 验证kernel和ramdisk签名
- 初始化dm-verity
- 设置SELinux策略
- 挂载system分区

**System验证**
- dm-verity实时验证
- 检测运行时篡改
- 处理验证错误

### AVB 2.0实现

**AVB元数据结构**
```
AvbVBMetaImageHeader {
    magic[4]: "AVB0"
    version_major: u32
    version_minor: u32
    authentication_data_block_size: u64
    auxiliary_data_block_size: u64
    algorithm_type: u32
    hash_offset: u64
    hash_size: u64
    signature_offset: u64
    signature_size: u64
    public_key_offset: u64
    public_key_size: u64
    ...
}
```

**分区验证方式**
1. **Hash验证**：适用于小分区（如boot）
2. **Hashtree验证**：适用于大分区（如system）
3. **Chain分区**：信任链传递

**vbmeta分区**
- 存储所有分区的验证元数据
- 包含公钥、签名、哈希值
- 支持A/B分区切换
- 防回滚保护信息

### dm-verity机制

Device-Mapper Verity提供透明的块设备完整性检查：

**工作原理**
```
应用读取 → VFS → dm-verity → 块设备
                      ↓
                 验证哈希树
```

**哈希树结构**
- 4KB数据块为叶子节点
- 每层哈希值组成上层节点
- 根哈希存储在vbmeta
- 支持按需验证

**错误处理模式**
1. **EIO模式**：返回I/O错误
2. **Restart模式**：重启设备
3. **Verify模式**：仅记录不处理

### 防回滚保护

**版本管理机制**
- Rollback Index：单调递增计数器
- 存储在防篡改存储（RPMB/独立芯片）
- 每个分区独立版本号
- OTA更新时自动更新

**回滚攻击防护**
```
if (image.rollback_index < stored_rollback_index) {
    // 拒绝启动，防止降级到有漏洞的版本
    boot_failed();
}
```

### 用户可控性

**设备锁定状态**
- **Locked**：普通用户模式，完整验证
- **Unlocked**：开发者模式，显示警告
- **Flashing**：刷机模式，暂时解锁

**解锁影响**
1. 清除用户数据（防止数据泄露）
2. 显示启动警告
3. 某些功能受限（如Netflix）
4. SafetyNet检测失败

### 与其他平台对比

**iOS Secure Boot**
- 完全封闭，用户无法解锁
- 硬件强制执行
- 更简单的信任链
- 与Secure Enclave深度集成

**Windows Secure Boot**
- UEFI标准实现
- 支持自定义密钥
- 主要防止bootkit
- 可完全禁用

**鸿蒙安全启动**
- 类似Android设计
- 增加远程认证
- 分布式设备信任
- 更严格的完整性要求

## 本章小结

本章深入探讨了Android密钥管理与硬件安全的核心机制：

1. **Keystore系统**提供了分层的密钥管理架构，从应用API到硬件安全模块，实现了密钥的安全生成、存储和使用。关键概念包括密钥生命周期管理、硬件密钥隔离、使用授权机制。

2. **Trusty TEE**作为可信执行环境，通过ARM TrustZone技术实现了与主系统的硬件隔离。Keymaster TA和Gatekeeper TA在TEE中提供关键的密码学服务和用户认证功能。

3. **硬件密钥认证**通过Key Attestation机制，生成包含设备安全状态的X.509证书链，使远程服务器能够验证密钥的硬件属性。StrongBox进一步提供了独立安全芯片级别的保护。

4. **安全启动链**通过Verified Boot确保从ROM到System的每一级启动组件都经过验证。dm-verity提供运行时的系统完整性保护，防回滚机制防止降级攻击。

关键公式和概念：
- 密钥派生：`KEK = KDF(HUK, User_Credential)`
- 认证链验证：`Verify(Cert[i], PublicKey[i+1])`
- 哈希树验证：`RootHash = Hash(Hash(Block1) || Hash(Block2) || ...)`
- 防回滚检查：`image.rollback_index ≥ stored_rollback_index`

## 练习题

### 基础题

**1. Keystore密钥存储位置**
Android Keystore中的密钥材料存储在哪里？软件密钥和硬件密钥的存储有何区别？

<details>
<summary>答案</summary>

软件密钥：
- 存储在/data/misc/keystore目录
- 使用keystore守护进程的主密钥加密
- 主密钥派生自用户凭据

硬件密钥：
- 密钥材料永不离开安全硬件（TEE/独立芯片）
- 系统只存储加密的key blob（句柄）
- 实际密钥操作在安全硬件中执行
</details>

**提示**：考虑密钥材料的可访问性和安全边界。

**2. TEE通信机制**
普通世界（REE）的应用如何与TEE中的可信应用通信？描述基本的通信流程。

<details>
<summary>答案</summary>

通信流程：
1. REE应用通过TEE Client API发起请求
2. TEE驱动准备共享内存缓冲区
3. 触发SMC（Secure Monitor Call）指令切换到安全世界
4. TEE OS接收请求并路由到目标TA
5. TA处理请求，结果写入共享内存
6. 返回普通世界，应用获取结果

关键组件：SMC指令、共享内存、TEE驱动
</details>

**提示**：思考CPU如何在两个世界之间切换。

**3. Key Attestation证书链**
描述Android Key Attestation生成的证书链结构，每个证书的作用是什么？

<details>
<summary>答案</summary>

证书链包含3个证书：
1. 密钥证书：包含被认证的公钥和认证扩展信息
2. 中间证书：设备特定的认证密钥证书，用于签名密钥证书
3. 根证书：Google硬件认证根证书，验证整个链的可信性

验证时从密钥证书开始，逐级验证签名直到可信的根证书。
</details>

**提示**：类比HTTPS证书链的验证过程。

### 挑战题

**4. 侧信道攻击防护**
设计一个密码学操作需要考虑哪些侧信道攻击？Android Keystore如何防护这些攻击？

<details>
<summary>答案</summary>

主要侧信道攻击类型：
1. 时序攻击：通过执行时间差异推断密钥
2. 功耗分析：DPA/SPA攻击
3. 电磁泄露：通过电磁辐射获取信息
4. 缓存攻击：利用缓存命中率差异

Android防护措施：
- 时间恒定算法实现（constant-time）
- 功耗随机化和屏蔽
- 硬件层面的电磁屏蔽（StrongBox）
- 密钥操作在隔离环境执行
- 添加随机延迟和虚假操作
</details>

**提示**：考虑攻击者可以观察到的物理特征。

**5. 安全启动绕过场景**
分析以下场景：攻击者物理接触设备并尝试刷入修改过的系统镜像。Verified Boot如何防止这种攻击？存在哪些可能的绕过方法？

<details>
<summary>答案</summary>

Verified Boot防护：
1. bootloader验证kernel签名
2. vbmeta包含所有分区哈希
3. 防回滚保护阻止旧版本
4. dm-verity运行时验证

可能的绕过方法：
1. 解锁bootloader（会清除用户数据）
2. 利用bootloader漏洞（如未正确验证）
3. 物理攻击提取OEM密钥
4. 供应链攻击（预装恶意固件）
5. 利用硬件debug接口

注意：正常解锁会触发数据清除，保护用户隐私。
</details>

**提示**：考虑攻击者的能力和系统的防护边界。

**6. 跨平台密钥迁移**
设计一个方案，允许用户在Android和iOS设备之间安全地迁移加密密钥。需要考虑哪些安全和技术挑战？

<details>
<summary>答案</summary>

方案设计要点：
1. 密钥导出限制：硬件密钥通常不可导出
2. 需要可信第三方或点对点加密通道
3. 临时迁移密钥的生成和销毁
4. 双向认证确保设备可信

技术挑战：
- 平台API差异（Keystore vs Keychain）
- 硬件安全模块不兼容
- 密钥属性映射（用途、算法限制）
- 时间窗口和重放攻击防护

可能方案：
1. 云端密钥托管（需信任云服务）
2. QR码+时限的密钥传输
3. 蓝牙/NFC近场加密传输
4. 派生密钥而非直接迁移
</details>

**提示**：硬件绑定的密钥设计初衷就是防止迁移。

**7. TEE漏洞利用分析**
假设在Trusty TEE的某个TA中发现了缓冲区溢出漏洞。分析这个漏洞的潜在影响和利用难度。

<details>
<summary>答案</summary>

潜在影响：
1. TA内权限提升
2. 访问其他TA的内存（如突破地址空间隔离）
3. 提取密钥材料
4. 持久化恶意代码

利用难度：
1. 需要先攻破REE获得TA调用能力
2. TEE通常启用ASLR/DEP/Stack Canary
3. 输入验证和大小限制
4. 缺乏调试能力增加开发难度
5. 设备差异化（不同TEE实现）

缓解措施：
- TA最小权限原则
- 形式化验证关键TA
- 运行时监控异常行为
- 定期更新TEE固件
</details>

**提示**：TEE设计目标是即使REE被攻破也能保护关键资产。

**8. 安全芯片设计权衡**
比较集成在SoC中的安全子系统（如ARM TrustZone）与独立安全芯片（如Titan M）的优缺点。在什么场景下应该选择哪种方案？

<details>
<summary>答案</summary>

SoC集成方案：
优点：
- 成本低，无需额外芯片
- 功耗优化好
- 带宽高，延迟低
- 软件栈成熟

缺点：
- 物理攻击防护弱
- 与主CPU共享资源
- 侧信道隔离有限

独立安全芯片：
优点：
- 物理防篡改设计
- 完全独立的计算环境
- 专用安全认证
- 更强的侧信道防护

缺点：
- 成本高
- 通信开销大
- 功耗额外负担
- 集成复杂度高

选择建议：
- 高价值设备→独立芯片
- 消费级设备→SoC集成
- 金融/政府应用→独立芯片
- IoT设备→依据成本权衡
</details>

**提示**：安全性、成本、性能的三角权衡。

## 常见陷阱与错误

1. **密钥泄露风险**
   - 错误：在日志中打印密钥相关信息
   - 错误：使用可导出密钥存储敏感数据
   - 正确：始终使用硬件密钥，设置正确的使用限制

2. **认证验证不当**
   - 错误：只验证证书链签名，不检查设备状态
   - 错误：信任客户端的认证结果
   - 正确：服务端完整验证，检查所有安全属性

3. **TEE通信错误**
   - 错误：在共享内存中传递敏感数据明文
   - 错误：未验证TA返回数据的完整性
   - 正确：加密敏感数据，使用HMAC保护完整性

4. **启动验证配置错误**
   - 错误：禁用dm-verity以提高性能
   - 错误：使用测试密钥签名生产镜像
   - 正确：保持完整验证链，妥善管理签名密钥

5. **密钥生命周期管理**
   - 错误：永不更新密钥
   - 错误：删除密钥前未备份重要数据
   - 正确：定期轮换，实现密钥恢复机制

## 最佳实践检查清单

- [ ] **密钥管理**
  - [ ] 使用硬件密钥存储敏感密钥
  - [ ] 设置适当的密钥使用限制
  - [ ] 实现密钥轮换策略
  - [ ] 正确处理密钥导入/导出

- [ ] **TEE集成**
  - [ ] 最小化TA的攻击面
  - [ ] 验证所有来自REE的输入
  - [ ] 使用安全的IPC机制
  - [ ] 定期更新TA版本

- [ ] **设备认证**
  - [ ] 服务端验证完整证书链
  - [ ] 检查设备安全状态
  - [ ] 使用足够随机的challenge
  - [ ] 实现认证失败的降级策略

- [ ] **安全启动**
  - [ ] 保持bootloader锁定
  - [ ] 启用所有验证功能
  - [ ] 监控启动失败事件
  - [ ] 及时应用安全更新

- [ ] **开发测试**
  - [ ] 使用独立的开发密钥
  - [ ] 测试密钥吊销流程
  - [ ] 验证错误处理路径
  - [ ] 进行安全审计
