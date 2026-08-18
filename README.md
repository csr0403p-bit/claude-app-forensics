# Claude Android App：设备信任与数据采集取证报告

> 静态反编译取证分析，回答三个问题：**服务端能拿到什么信息、如何拿到、在什么情况下拿到**——以及每条信息意味着什么，和"大陆账号 + VPN 仍被封"现象的原理推测。
>
> - 对象：`com.anthropic.claude` v1.260813.10（Play 商店版）
> - 方法：jadx 1.5.6 反编译 + dex 字符串表 + AXML 二进制解析（AndroidManifest / 备份规则）
> - 日期：2026-08-18
> - 性质：纯静态分析，未对运行中的 app 或服务端做任何探测
> - 源码根目录：APK 反编译产物 `jadx-out/sources/`（正文所有路径均相对此根；R8 混淆类位于 `defpackage` 包）

---

## 目录

1. [执行摘要：三条采集通道](#1-执行摘要三条采集通道)
2. [已证实的信息链路总表](#2-已证实的信息链路总表)
3. [通道① Play Integrity：登录必发的设备证明](#3-通道-play-integrity登录必发的设备证明)
4. [通道② 受信设备：硬件密钥与每消息签名](#4-通道受信设备硬件密钥与每消息签名)
5. [通道③ Sift：启动即采的第三方设备指纹](#5-通道-sift启动即采的第三方设备指纹)
6. [第四通道（服务端视角）：ActorSnapshot 参与者快照](#6-第四通道服务端视角actorsnapshot-参与者快照)
7. [本地身份残留：登出清不掉的证据](#7-本地身份残留登出清不掉的证据)
8. [受信设备完整时序](#8-受信设备完整时序)
9. [每个数据点意味着什么](#9-每个数据点意味着什么)
10. [封禁原理推测：大陆账号 + VPN 仍被封](#10-封禁原理推测大陆账号--vpn-仍被封)
11. [方法与局限](#11-方法与局限)
12. [附录 A：关键混淆类速查表](#12-附录-a关键混淆类速查表)
13. [附录 B：关键字符串清单](#13-附录-b关键字符串清单)
14. [附录 C：源码位置索引](#14-附录-c源码位置索引)

---

## 1. 执行摘要：三条采集通道

app 对设备的认定不靠一条链路，而是三条独立通道 + 一个服务端快照字段，互为补充——这也是"换账号 / 换 VPN"难以规避的原因。

| 通道 | 触发时机 | 门控（开关） | 核心能力 |
|---|---|---|---|
| ① Play Integrity | 每次登录 / 魔法链接验证 | **无**（登录必发） | 向 Google 请求"真实设备、未被篡改"的签名回执 |
| ② 受信设备 | 服务器 403 触发登记；会话过期触发 reattest | `sessions_elevated_auth_enforcement`、`ccr_step_up_reauth`、`mobile_ccr_per_message_attestation`、`ccr_attestation_self_heal` | 安全芯片私钥 + 每消息签名，物理级设备身份 |
| ③ Sift | App 启动（首个 Activity 创建）即采集 | 组织策略 `third_party_analytics_disabled_for_org`、`sift_extended_beacon`（定位） | 设备指纹（android_id、SIM、网卡 IP、root 证据等）批量上报 |
| ④ ActorSnapshot | 服务端侧 actor 身份字段（推测为随 API 请求自动注入） | 不可见 | `client_ip` / `user_agent` / `device_id` / `client_platform` |

---

## 2. 已证实的信息链路总表

以下每条均有反编译代码证据，证据位置见各章展开。

| # | 服务端拿到的信息 | 采集通道 | 触发条件 | 能否被 VPN 影响 |
|---|---|---|---|---|
| 1 | Play Integrity 完整性 token | ① | 每次登录，无开关 | 否 |
| 2 | 设备公钥 + device_id + device_token | ② 登记 | 服务器 403 时自动触发 | 否 |
| 3 | 每消息设备签名（Id/Timestamp/Signature 头） | ② | 登记成功后所有消息 | 否 |
| 4 | android_id | ③ | 启动即采 | 否（恢复出厂才变） |
| 5 | SIM 运营商名 + 国家码 | ③ | 启动即采 | 否（SIM 是实体） |
| 6 | 全部非回环网卡 IP | ③ | 启动即采 | 否（VPN 只换出口 IP） |
| 7 | 设备定位（经纬度） | ③ | `sift_extended_beacon`=true 且已授予定位权限 | 否 |
| 8 | root / 改机 / 模拟器证据 | ③ | 启动即采 | 否 |
| 9 | 设备型号 / 厂商 / 系统版本 / SDK | ③ | 启动即采 | 否 |
| 10 | Claude 账号 ID ↔ 设备指纹绑定 | ③（Sift userId=AccountId） | 登录后，登出后仍残留 | 否 |
| 11 | 登录会话 cookie（明文文件名含账号 ID） | 登录 | 登录后持续 | 否 |
| 12 | 最近登录账户 UUID / 历史账户集合 | 本地写入 | 登录后；**登出不清；重装经自动备份复活** | 否 |
| 13 | Firebase 设备 ID（FID）与刷新令牌 | Firebase Installations | 启动；登出不清 | 否 |
| 14 | 设备生物识别绑定状态 | ② reattest 签名验证 | 会话过期触发 reattest 时 | 否 |
| 15 | 出口 IP | 服务器网络层 | 每次请求 | **是（唯一可变的）** |

---

## 3. 通道① Play Integrity：登录必发的设备证明

### 3.1 请求结构（字段级）

魔法链接验证请求 `VerifyMagicLinkRequest` 共 6 个字段，序列化定义于
`com/anthropic/claude/api/login/d.java:27-32`：

| 字段 | 序列化名 | required | 说明 |
|---|---|---|---|
| `credentials` | `credentials` | 是 | 邮箱登录凭证 |
| `recaptcha_token` | `recaptcha_token` | 否 | reCAPTCHA（两个均非必填） |
| `recaptcha_site_key` | `recaptcha_site_key` | 否 | 同上 |
| `source` | `source` | 否 | 默认值 `"claude"` |
| `play_integrity_token` | `play_integrity_token` | 是 | **Play Integrity 证明，必填** |
| `client_attestation` | `client_attestation` | 是 | 包装类型，见下 |

`client_attestation` 包装类型 `com/anthropic/claude/api/login/ClientAttestation.java:40`：
字段仅一个，构造器签名 `ClientAttestation(play_integrity_token=...)`——即把 token 包进
`client_attestation` 对象随登录请求提交。

### 3.2 实现细节（defpackage/uga.java，PlayIntegrityManager）

流程：

1. 构造 `IntegrityTokenRequest`（经 `cloudProjectNumber` 标识项目）请求完整性 token；
2. **Play Store 版本门槛**：读取 `com.android.vending` 的版本码，低于 `82380000` 直接失败
   （对应 Play 商店服务端强制升级策略的客户端侧预检）；
3. 等待 token 生成，**超时 3 秒**；
4. 成功后将 token 附到登录请求的 `client_attestation` 中。

调用方：

- `defpackage/m2c.java:31` —— 邮箱登录 ViewModel；
- `defpackage/z2c.java` —— 魔法链接登录 ViewModel（经 `a79` 拉取 token）；
- `defpackage/i8.java` —— 另一处登录入口。

### 3.3 门控结论

在实验开关注册表（`defpackage/pg9.java`，约 100 个 gate）中**没有找到任何控制
Play Integrity 的开关**——登录必然携带，无服务器遥控关闭的余地。

### 3.4 服务端能得到什么

- Google 对"设备真实性与完整性"的签名判定：真实设备 / 模拟器 / root / 应用被篡改；
- 该 token 与 Google Play 服务身份链绑定，可跨账号关联"同一台设备上的 Google 身份"；
- 对模拟器、改机软件、养号环境构成第一道物理拦截。

---

## 4. 通道② 受信设备：硬件密钥与每消息签名

### 4.1 总览

- 协调器：`defpackage/zsk.java`（`TrustedDeviceCoordinator`，约 2070 行）；
- 密钥管理：`defpackage/d5g.java`（`ReattestKeyProvider`）；
- 偏好存储：`defpackage/btk.java`。

### 4.2 四个端点（Retrofit 接口）

| 端点 | 接口定义 | 用途 |
|---|---|---|
| `POST auth/trusted_devices` | `defpackage/gsk.java` | 登记设备 |
| `POST auth/trusted_devices/{device_id}/rotate_reattest` | `defpackage/gsk.java` | 轮换 reattest 密钥 |
| `GET auth/session_reattest/device_key/challenge` | `defpackage/q4i.java` | 获取 reattest 挑战 |
| `POST auth/session_reattest/device_key` | `defpackage/q4i.java` | 提交挑战签名 |

### 4.3 DTO 字段（com/anthropic/claude/api/trusteddevice/）

```java
EnrollTrustedDeviceRequest(display_name, device_public_key, reattest_public_key, platform)
  // platform 固定 "android"
EnrollTrustedDeviceResponse(device_id, device_token, reattest_kid, device_binding_kid)
ReattestChallengeResponse(challenge_id, challenge, expires_at?)
ReattestVerifyRequest(challenge_id, trusted_device_id, signature, kid)
RotateReattestRequest(new_reattest_public_key, device_binding_timestamp,
                      device_binding_signature, predecessor_kid, device_binding_kid,
                      platform, attestation_blob)
RotateReattestResponse(...)
```

### 4.4 密钥生成（硬件级，defpackage/d5g.java）

```java
KeyPairGenerator.getInstance("EC", "AndroidKeyStore")
// 曲线 secp256r1，签名算法 SHA256withECDSA
// StrongBox 优先：
//   setIsStrongBoxBacked(true) → 失败回退 TEE，
//   日志: "StrongBox keygen failed … falling back to TEE"
```

两个密钥别名，用途不同：

| 别名 | 生物识别绑定 | 用途 |
|---|---|---|
| `trusted_device_binding` | 否 | 设备登记的"根"密钥；sign rotate 材料 |
| `trusted_device_reattest` | **是**（`setUserAuthenticationRequired(true)` + `setInvalidatedByBiometricEnrollment(true)`） | 会话 reattest；**改指纹/面容令其作废** |

私钥永驻硬件，不进任何文件；app 只持有公钥与证书链。

### 4.5 登记触发条件（何时才会登记）

1. 服务器返回 403，错误码含 "Elevated auth required"，或返回
   `permission_denied` / `permission_error` 且消息含 "trusted device"；
2. 错误映射（`defpackage/ijn.java:471-502`，另一处 `ij1.java:1065-1092`）→
   枚举 `yi7.UNTRUSTED_DEVICE`；
3. 门卫 `defpackage/cj7.java` 启动登记流程——**该门卫仅在服务器标志
   `sessions_elevated_auth_enforcement` 开启时创建**
   （`com/anthropic/claude/code/remote/h.java:236-243`）；
4. 编排类 `defpackage/ccn.java` 执行；组包在 `defpackage/g8.java:250-282`（case 7）：
   `EnrollTrustedDeviceRequest(display_name="Claude on …", device_public_key, reattest_public_key, platform="android")`。

### 4.6 reattest（会话过期时的重新证明）

1. 服务器返回 `SESSION_STALE_RELOGIN`；
2. 门控 `zsk.e()` 五查（`zsk.java:400-418`）：
   - 开关 `ccr_step_up_reauth` = true；
   - 生物识别可用（`BiometricManager`）；
   - reattest 密钥存在（KeyStore 别名）；
   - device_id 存在（btk prefs）；
   - （以上全通过才继续）
3. 弹系统 `BiometricPrompt`，用绑定生物识别的硬件密钥签名；
4. 签名材料（`defpackage/vsk.java`，case 1）：

   ```
   GET
   api/auth/session_reattest/device_key/challenge
   <epochSecs>
   ```

5. 提交 `ReattestVerifyRequest(challenge_id, trusted_device_id, signature, kid)`
   （组包 `defpackage/ok5.java`，case 3）。

### 4.7 每消息认证头（登记成功后的常态）

签名前缀：`"anthropic.reattest.v1\0"`（`defpackage/i5g.java`）。

| 请求头 | 内容 |
|---|---|
| `X-Trusted-Device-Id` | device_id |
| `X-Trusted-Device-Timestamp` | 时间戳 |
| `X-Trusted-Device-Signature` | 硬件密钥签名 |
| `X-Trusted-Device-Token` | device_token（由拦截器 `defpackage/m9i.java` / `defpackage/gk1.java` 自动附加） |

服务器据此可对**每一条消息**验证"这来自登记过的那台物理设备"。

### 4.8 轮换（rotate）

- 挑战材料：`SHA-256(会话 UUID)`；
- 签名材料（`defpackage/o87.java`）：`"anthropic.reattest_rotate.v1\0"` +
  8 字节大端时间戳 + 新 reattest 公钥；
- `attestation_blob` = `KeyStore.getCertificateChain` 证书链拼接（`defpackage/tsk.java`）；
- 组包 `RotateReattestRequest(..., predecessor_kid, device_binding_kid, platform, attestation_blob)`
  （`defpackage/ok5.java`，case 3 之外另一分支）；
- 由 `zsk.p()` 协调，通常伴随 reattest 一并发生。

### 4.9 自愈（Heal）

- 前置：`ccr_attestation_self_heal` 且 `mobile_ccr_per_message_attestation` 均为真；
- 预算：`heal_enroll_attempts < 3`（btk prefs 计数）；
- 触发日志：`"Reattest key unusable on enrolled device; attempting post-login heal"`；
- 行为：密钥失效（如生物识别变更）或丢失时，登录后自动重新登记。

### 4.10 Cookie 与本地存储

- Cookie：`__Host-ant_trusted_device`，由 `defpackage/v9h.java`（CookieJar，
  `SerializableCookie` JSON 序列化）持久化；
- 偏好存储：加密 SharedPreferences `"trusted_device"`（`defpackage/btk.java` keys：
  `token` / `device_id` / `reattest_kid` / `device_binding_kid` / `heal_enroll_attempts`）；
- 加密实现（`defpackage/ln7.java` / `nn7.java`）：AES256-SIV/GCM，主密钥存
  AndroidKeyStore——即明文不可直接读取，但 app 自己有解密路径。

### 4.11 登出清理（e.java）

- `e.java:1437` —— "Clear trusted-device token + reattest keys" 处理器：
  `btk.a(false)`（清 prefs）+ `d5g.a()`（删 KeyStore 密钥）+ `v9h.e()`（清 cookie）+
  `zsk.o()`（重置协调器状态）；
- `e.java:1485` —— stranded-kid（有密钥无 token 的残留 kid）清理；
- `e.java:1528` —— 密钥存在但无 token 时的重新登记路径。

---

## 5. 通道③ Sift：启动即采的第三方设备指纹

### 5.1 初始化（非自动注册，手动生命周期采集）

```java
// defpackage/hli.java:35 —— ActivityLifecycleCallbacks.onActivityCreated
Sift.open(activity, Config.Builder()
    .withAccountId(ACCOUNT_ID)
    .withBeaconKey(BEACON_KEY)
    .withDisallowLocationCollection(!sift_extended_beacon)   // 定位受 gate 控制
    .build());
Sift.collect();
```

注册点：`com/anthropic/claude/application/ClaudeApplication.java:248,263`。

### 5.2 生产凭据与端点

- 生产 accountId：`64e6742e35ba4d3981f27c05`，beaconKey：`99dfa2e716`
  （`defpackage/ili.java`；沙箱凭据在 `defpackage/jli.java`）；
- 端点（`siftscience/android/Sift.java:149`）：

  ```
  PUT https://api3.siftscience.com/v3/accounts/{accountId}/mobile_events
  ```

- 认证：HTTP Basic，凭据为 base64(beaconKey)；
- 传输：gzip；失败重试最多 3 次，退避 3s / 12s。

### 5.3 采集字段（字段名来自 JSON 定义）

设备属性（`siftscience/android/DevicePropertiesCollector.java`）：

- `android_id` —— `Settings.Secure.ANDROID_ID`，同时作为 Sift `installationId`；
- `mobile_carrier_name` / `mobile_iso_country_code` —— `TelephonyManager`（**SIM 层信号**）；
- `device_manufacturer` / `device_model` / `device_system_version` / `sdk_version` / `build_tags`；
- `app_name` / `app_version`；
- root 证据（`evidence_*` 系列）：
  - 10 个常见 SU 路径的存在性检查（`SU_PATHS`）；
  - 已知 root / 危险包名检测；
  - `getprop` 读取 `ro.debuggable`（=1 可疑）、`ro.secure`（=0 可疑）；
  - `mount` 检查 `/system` 是否 rw。

App 状态（`siftscience/android/AppStateCollector.java`）：

- `network_addresses` —— **全部非回环网卡接口的 IP**（IPv4/IPv6）；VPN 只改出口路由，改不掉本机接口地址；
- 电池 / 充电状态；
- 定位 —— Google FusedLocationClient → `AndroidDeviceLocationJson`（受 gate + 权限双重门控）。

**取证修正**：`installed_apps` 字段存在于 `com/sift/api/representations/AndroidDevicePropertiesJson.java`
的 schema 中，但采集器**从未填充**——"App 会读取你安装了哪些应用"不成立。

### 5.4 队列与上传时机

- 设备属性队列：有事件立即上传，同类事件 1 小时去重；
- App 状态队列：攒满 8 条，或距上次上传超过 60 秒；
- 本地缓存：SharedPreferences `"siftscience"`（`siftscience.xml`：config / user_id / 队列），
  进程重启后恢复并续传；
- enabled 状态：`defpackage/lli.java`。

### 5.5 账号绑定

- Sift `userId` 直接设为 **Claude AccountId**：`defpackage/wal.java:47-49`
  （`Sift.updateUserId(accountId)`），`defpackage/fhk.java:55`；
- 即 Sift 平台侧维护 "账号 → 设备指纹" 映射，与 Anthropic 服务端数据可交叉关联。

### 5.6 开关

| 开关 | 控制点 | 位置 |
|---|---|---|
| `third_party_analytics_disabled_for_org`（组织级） | 禁用全部三方分析（Datadog/Segment/Firebase/Sentry/Sift） | `defpackage/j6k.java`（`ThirdPartyAnalyticsGate`）→ `defpackage/pp3.java` case 19 `setEnabled` |
| `sift_extended_beacon` | 定位采集（默认开） | `defpackage/go0.java:34` |

---

## 6. 第四通道（服务端视角）：ActorSnapshot 参与者快照

`com/anthropic/bard/api/v1alpha/ActorSnapshot.java`（proto 定义，`v1alpha` 表明是服务端
actor 身份模型）：

```proto
message ActorSnapshot {
  string client_ip = ...;
  string user_agent = ...;
  string device_id = ...;
  string client_platform = ...;
}
```

- 这是**服务端侧**的参与者身份模型：`client_ip` 由服务端从请求连接取得，
  `user_agent` / `device_id` / `client_platform` 由客户端声明；
- 表明服务端在 API 层维护统一的 actor 身份快照——与受信设备体系互相印证：
  设备 ID 不是孤立的本地概念，而是服务端会话模型的一等公民。

---

## 7. 本地身份残留：登出清不掉的证据

### 7.1 登出会清掉的（AccountLogoutCleaner：`defpackage/i8.java` / `h8.java`）

| 项 | 位置 | 清理动作 |
|---|---|---|
| 受信设备凭证 | `shared_prefs/trusted_device.xml`（AES256-SIV/GCM 加密） | 清 prefs |
| KeyStore 私钥 | 别名 `trusted_device_binding` + `trusted_device_reattest` | 删密钥 |
| 全部登录 cookie | `user_cookies_login.xml` / `user_cookies_<accountId>.xml`（明文） | 清 cookie |
| Firebase 分析库 | `google_app_measurement.db`（+ wal/shm/journal） | 删 DB（`defpackage/n9.java`） |
| Credential Manager passkey | 系统凭据 | 删除 |
| 调试预存 | debug prefs | 删除 |

### 7.2 登出后仍然残留的

| 项 | 位置 | 说明 |
|---|---|---|
| **Sift user_id + 事件队列** | `siftscience.xml`（明文 JSON） | `unsetUserId()` 只改内存不写文件——文件中的 user_id（= 账号 ID）与未上传队列原样保留 |
| **最近登录账户 UUID** | `app_prefs.xml` → `app_last_account_id` | 当前账号登出时会被移除并轮换，但**无任何专门清除路径** |
| **历史账户集合** | `app_prefs.xml` → `app_known_account_ids` | 其余账户的 ID 保留 |
| **`has_logged_in_before`** | `app_prefs.xml` | 置为 true 后**永不清除** |
| `last_subscription_level` / `is_ant` | `app_prefs.xml` | 订阅与机构标记 |
| Firebase FID + 刷新令牌 | `files/PersistedInstallation.<appId>.json` | 登出不清（`defpackage/npe.java:633-648`） |

### 7.3 重装后从云端备份复活的

Android 自动备份白名单（`backup_rules.xml` / `data_extraction_rules.xml`）**只包含两个文件**：

```
app_prefs.xml      ← 最近/历史账户 ID（身份核心）
app_stats.xml      ← 使用统计（无身份）
```

- 即：**跨重装复活的是"这台设备用过哪些账号"**；
- 受信设备凭证、cookie、Sift 文件、Firebase 均不在白名单内，重装即失。

### 7.4 任何应用级操作都清不掉的

- `android_id`（`Settings.Secure`，系统级，仅恢复出厂或 Android 8+ 各签名 key 重置会变）；
- 服务器端已记录的全部信息（黑名单、指纹库、封禁记录）。

---

## 8. 受信设备完整时序

```
[触发] 用户发消息 → 服务器 403（elevated auth required / permission_denied + "trusted device"）
   → 错误映射 yi7.UNTRUSTED_DEVICE（ijn.java:471-502）
   → 门卫 cj7（仅 sessions_elevated_auth_enforcement 开启时存在，h.java:236-243）
        │
[铸钥] AndroidKeyStore：EC secp256r1（StrongBox → TEE 回退，d5g.java）
   → 两把密钥：trusted_device_binding（无生物识别）/ trusted_device_reattest（绑定生物识别）
        │
[登记] POST auth/trusted_devices（g8.java:250-282 组包）
   → 响应：device_id / device_token / reattest_kid / device_binding_kid
   → 或经 Set-Cookie 下发 __Host-ant_trusted_device
   → 加密写入 trusted_device.xml（btk.java）
        │
[日常] 每条消息自动附加 X-Trusted-Device-Id/Timestamp/Signature/Token（i5g/m9i/gk1）
        │
[会话过期] SESSION_STALE_RELOGIN
   → zsk.e() 五重门控（ccr_step_up_reauth / 生物识别 / 密钥 / device_id，zsk.java:400-418）
   → BiometricPrompt → 硬件密钥签名挑战（vsk.java 签名材料）
   → POST auth/session_reattest/device_key（ok5.java case 3）
        │
[轮换] 伴随 reattest：challenge = SHA-256(会话 UUID)
   → binding 密钥签 "anthropic.reattest_rotate.v1\0" + 时间戳 + 新公钥（o87.java）
   → attestation_blob = KeyStore 证书链（tsk.java）
   → POST auth/trusted_devices/{device_id}/rotate_reattest
        │
[自愈] 密钥失效/丢失 → 登录后 heal（ccr_attestation_self_heal && mobile_ccr_per_message_attestation，
       heal_enroll_attempts < 3）
```

---

## 9. 每个数据点意味着什么

单条信息都很普通，但组合起来就是一台设备剥不掉的数字指纹——**让服务器能认出
"这依然是同一台手机、同一个人"**。

| 数据点 | 服务器拿它能做什么 | 人话含义 |
|---|---|---|
| `android_id` | 跨账号、跨登录、跨登出识别同一台设备 | 设备的"身份证号"。换账号、换 VPN 都甩不掉，唯一重置方式是恢复出厂。服务器由此把"这台手机"与所有登录过它的账号编成一张表 |
| 运营商 + SIM 国家码 | 判断设备（及其使用者）物理所在国家/地区 | SIM 卡是实体的——插着中国移动的卡，服务器就知道这台手机（及大概率的人）在中国。VPN 管不到 SIM |
| 全部网卡 IP | 拿到设备本地网络真实地址，与出口 IP 对照 | VPN 只换"出口 IP"，app 却把本机网卡 IP 也报上去。两个 IP 一对："流量从美国来，但设备物理上在中国家里"——肉身与 IP 分离一眼可见 |
| 设备定位 | 直接确认物理位置 | 权限与开关都开时，这是"人在中国"最确凿的证据，任何网络手段绕不开 |
| 设备型号 / 系统 / build | 与其他信号组成指纹组合 | 单看无意义，组合能识别"同一台手机"，也能发现"同型号批量注册"的养号行为 |
| root / 改机证据 | 标记异常设备，降级信任 | 反欺诈平台对这类设备基本不信任，风险分起手就高 |
| Play Integrity 回执 | 确认真实设备 + app 未被篡改 + 绑定 Google 账户链 | 模拟器与改机登录第一步现形；同时这台设备连同其 Google 身份成为可追踪关联点 |
| 受信设备 device_id / 公钥 / 签名 | 物理级身份，每条消息可验证 | "换账号没用"的根源——账号可注册一百个，芯片签名的章只有这台手机盖得出 |
| 账号 ID ↔ 设备指纹绑定（Sift userId） | 建立"哪个账号在用哪台手机"的映射 | 服务器一张对照表：登录即知"账号 A 与账号 B 用的是同一台手机" |
| `app_last_account_id` / `app_known_account_ids` | 登出后仍记录这台手机用过谁；重装经备份复活 | 你以为登出就干净了，"最近登录过哪些账号"留在 `app_prefs.xml`，换账号后历史身份依然可关联 |
| Firebase FID + 刷新令牌 | 绕过 cookie 的另一个设备级锚点 | cookie 清了它还在，同样是设备关联的身份文件 |
| `installed_apps`（已安装应用列表） | —— | **修正结论：不采集**。字段存在但从未被填充；应用是否被篡改由 Play Integrity 检查 |

---

## 10. 封禁原理推测：大陆账号 + VPN 仍被封

> **本节为推测。** 服务端决策逻辑（评分规则、阈值、黑名单内容）不在 APK 内。
> 以下基于第 2–7 节已证实的客户端证据 + 通用反欺诈行业实践推断，每一环标注依据。

### 10.1 事实与推测的边界

| 事实（客户端代码可见） | 推测（不可见，行业惯例推断） |
|---|---|
| 服务器能拿到 5 个"大陆身份信号"，其中 4 个与 VPN 无关（SIM 国家码、运营商、本机网卡 IP、设备定位） | 服务器如何组合这些信号 |
| 设备有 3 层持久身份可跨账号关联（`android_id`、受信设备硬件密钥、`app_last_account_id`/FID 残留） | 评分阈值定在哪里 |
| 每消息带硬件签名、cookie、指纹上报全链路齐备 | 封禁由哪个环节触发 |

### 10.2 推断的封禁链条

1. **反欺诈普遍用"风险评分"而非单信号判定。**（行业实践）任何单信号都有大量误报——
   出差的人、留学生、用机场节点的普通人——通行做法是把信号加权求和，超过阈值才动手。
2. **大陆信号贡献高权重。**（第 2 节 #5/#6/#7 行）SIM 国家码 = CN、运营商为三大运营商、
   本机网卡 IP 落在中国网段、定位在中国 → "设备物理上在中国"几乎可以确定。
3. **VPN 只减一项分。**（第 2 节 #15 行）出口 IP 变化只影响网络层那一项，其余各项纹丝不动，
   总分照样过线——这就是"挂了 VPN 还是被封"的机制层面解释。
4. **封禁后设备指纹入库。**（第 4/5/7 节）android_id、受信设备密钥、网卡 IP、账号残留全部
   进入黑名单库。这是"同设备换新账号立刻再被封"的最合理解释。
5. **设备级命中优先于账号级判定。**（推断）新账号登录的瞬间，设备指纹先撞库 → 直接封。
   所以"新注册账号"与"旧账号"在同一台手机上下场一样。
6. **月卡批次连坐（连环封主因）。**（推断，支付反欺诈惯例）"月卡账号"大概率出自同一支付
   批次——同一张卡、同一个代充渠道。支付方判定欺诈或退款时，整批账号被标记。这是
   **账号维度**的一刀切：换设备、换 IP、换指纹全部无效，因为封的是账号本身。

### 10.3 为什么 VPN 没用：7 个信号里它只能改 1 个

| 大陆身份信号 | 采集方式（已证实） | VPN 能改变？ |
|---|---|---|
| 出口 IP | 服务器网络层 | ✓ 唯一能改的 |
| SIM 国家码 / 运营商 | Sift 启动即采（TelephonyManager） | ✗ SIM 卡是实体 |
| 本机网卡 IP | Sift `network_addresses` | ✗ 只有出口 IP 变 |
| 设备定位 | Sift FusedLocation（gate + 权限） | ✗ 只能拒绝权限 / 关定位 |
| 设备历史记录 | 服务器黑名单（此前封禁入库） | ✗ 已在库里 |
| `android_id` | Sift 启动即采 | ✗ 需恢复出厂 |
| 受信设备密钥 | 服务器已登记（第 4 章） | ✗ 需登出 + 清数据 + 重铸 |

### 10.4 结论

封禁发生在**登录那一刻的静态信号**上，与你之后发什么消息无关——这也解释了
"刚注册就被封、没说过一句话"的现象。VPN 能改变的出口 IP 只是 7 个信号中的 1 个；
"用了 VPN 还是被封"不是 VPN 没生效，而是它本来就不在这些信号的覆盖范围内。

### 10.5 这条链条的两处软肋

- 步骤 1 的"评分制"是行业惯例推断，Anthropic 也可能用更简单的规则（如"国家码 = CN 直接拒绝"）；
- 步骤 6 的批次连坐依赖支付渠道的反欺诈联动，无法从 APK 证实。

若将来获得更多信息（服务端行为观测、支付渠道反馈），这两环需要重新评估。

---

## 11. 方法与局限

- **纯静态分析**：jadx 1.5.6 反编译 APK + dex 字符串表 + AXML 二进制解析
  （AndroidManifest / 备份规则）。未对运行中的 app 或服务端做任何探测。
- **混淆还原**：R8 混淆（单字母类名，`defpackage` 包）；部分方法因指令量过大未被
  jadx 还原（如 `zsk.l()` 828 条指令、登出协程 `h8` 2676 条指令），相关结论由
  字段类型 + 字符串 + 唯一调用方法三重交叉佐证。
- **服务端不可见**：封禁规则、匹配阈值、黑名单内容不在 APK 内。所有涉及服务端行为的
  表述均为基于采集端的合理推断，已在正文标注。
- **版本时效**：基于 v1.260813.10；app 的风控能力经实验开关（gates）逐步放量，
  后续版本可能调整。
- **一处修正**：早期分析曾认为 Sift 采集已安装应用列表；代码核验后发现
  `installed_apps` 字段未被采集器填充，本报告以修正后的结论为准。

---

## 12. 附录 A：关键混淆类速查表

| 类 | 职责 | 关键位置 |
|---|---|---|
| `zsk` | TrustedDeviceCoordinator：登记/reattest/轮换编排、门控 | `zsk.java:400-418`（五重门控）、`zsk.e()`/`zsk.p()`/`zsk.o()` |
| `d5g` | ReattestKeyProvider：KeyStore 密钥生成 | `d5g.java:206-214`（生物识别绑定） |
| `g8` | 登记请求组包 | `g8.java:250-282`（case 7） |
| `ok5` | reattest / rotate 请求组包 | case 3（verify） |
| `vsk` | 挑战签名材料构造 | case 1：`GET\napi/auth/session_reattest/device_key/challenge\n<epochSecs>` |
| `o87` | rotate 签名材料构造 | `anthropic.reattest_rotate.v1\0` |
| `i5g` | 每消息签名前缀 | `anthropic.reattest.v1\0` |
| `tsk` | attestation_blob（Keystore 证书链） | —— |
| `m9i` / `gk1` | 请求拦截器（附加设备头/token） | —— |
| `cj7` | 登记门卫（elevated auth） | —— |
| `ccn` | enroll 编排 | —— |
| `btk` | trusted_device 偏好存储 | keys：token/device_id/reattest_kid/device_binding_kid/heal_enroll_attempts |
| `v9h` | CookieJar（SerializableCookie） | `__Host-ant_trusted_device` |
| `e` | 设置页/登出处理器 | `e.java:1437`（清 trusted-device）、`1485`（stranded-kid）、`1528`（re-enroll） |
| `uga` | PlayIntegrityManager | Play Store ≥ 82380000、3s 超时 |
| `m2c` / `z2c` | 登录 ViewModel | `m2c.java:31` |
| `hli` | Sift 生命周期初始化 | `hli.java:35` |
| `lli` | Sift enabled 状态 | —— |
| `ili` / `jli` | Sift 生产/沙箱凭据 | accountId `64e6742e35ba4d3981f27c05` |
| `j6k` | ThirdPartyAnalyticsGate | `third_party_analytics_disabled_for_org` |
| `pp3` | gate 应用分发 | case 19（三方分析 setEnabled） |
| `go0` | gate 读取 | `go0.java:34`（`sift_extended_beacon`） |
| `pg9` / `kg9` / `ca9` | 自研 GrowthBook gates 注册表 | 约 100 个 gate |
| `ln7` / `nn7` | 加密 SharedPreferences | AES256-SIV/GCM，keystore 主密钥 |
| `im0` | app_prefs 键定义 | `im0.java:325-333`（app_last_account_id 等） |
| `i8` / `h8` | AccountLogoutCleaner | 字段清单见第 7 节 |
| `npe` | PersistedInstallation JSON | `npe.java:633-648` |
| `n9` | Firebase 分析 DB 删除 | `google_app_measurement.db` |
| `q4i` / `gsk` | Retrofit 端点接口 | 第 4.2 节 |
| `ij1` / `ijn` | 错误映射 | `ij1.java:1065-1092`、`ijn.java:471-502` |
| `yi7` | UNTRUSTED_DEVICE 枚举 | —— |
| `wal` / `fhk` | Sift userId = AccountId | `wal.java:47-49`、`fhk.java:55` |
| `h` | 远程配置/门卫工厂 | `com/anthropic/claude/code/remote/h.java:236-243`（sessions_elevated_auth_enforcement） |

---

## 13. 附录 B：关键字符串清单

```
# 受信设备端点（dex_strings.txt）
auth/trusted_devices
auth/trusted_devices/%s/rotate_reattest
auth/session_reattest/device_key
auth/session_reattest/device_key/challenge

# Cookie
__Host-ant_trusted_device

# 签名前缀
anthropic.reattest.v1\0
anthropic.reattest_rotate.v1\0

# 错误码 / 触发条件
SESSION_STALE_RELOGIN
Elevated auth required
permission_denied / permission_error + "trusted device"

# 实验开关（gates）
ccr_step_up_reauth
ccr_attestation_self_heal
mobile_ccr_per_message_attestation
sessions_elevated_auth_enforcement
sift_extended_beacon
ccr_initial_events_limit        # default 200
third_party_analytics_disabled_for_org

# Sift
api3.siftscience.com/v3/accounts/%s/mobile_events
64e6742e35ba4d3981f27c05       # 生产 accountId
99dfa2e716                      # beaconKey

# 日志锚点
"Reattest key unusable on enrolled device; attempting post-login heal"
"StrongBox keygen failed … falling back to TEE"
```

---

## 14. 附录 C：源码位置索引

```
com/anthropic/claude/api/login/d.java              — VerifyMagicLinkRequest 序列化器
com/anthropic/claude/api/login/ClientAttestation.java — play_integrity_token 包装
com/anthropic/claude/api/trusteddevice/            — 6 个 DTO（Enroll/Reattest/Rotate）
com/anthropic/claude/sessions/types/DeviceAttestation.java — (kid, signature)
com/anthropic/bard/api/v1alpha/ActorSnapshot.java  — client_ip/user_agent/device_id/client_platform
com/anthropic/claude/application/ClaudeApplication.java — Sift 生命周期注册（248,263）
com/anthropic/claude/code/remote/h.java            — 门卫工厂（236-243）
siftscience/android/Sift.java                      — 上传端点（149）
siftscience/android/DevicePropertiesCollector.java — android_id/carrier/root 证据
siftscience/android/AppStateCollector.java         — network_addresses/battery/location
siftscience/android/SiftImpl.java                  — 队列与 SharedPreferences "siftscience"
com/sift/api/representations/AndroidDevicePropertiesJson.java — 字段定义（installed_apps 未填充）
defpackage/                                       — 全部 R8 混淆类（见附录 A）
```

---

*本报告为静态取证分析，仅供技术研究。对服务端行为的表述均为基于客户端证据的推断。*
