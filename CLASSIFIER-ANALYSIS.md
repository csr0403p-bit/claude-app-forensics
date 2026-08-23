# Claude Android App 分类器定位报告：分类器在 app 里吗？

> 核心问题：**内容安全分类器（审核/过滤模型）是否运行在 app 内？**
>
> - 对象：`com.anthropic.claude` **v1.260817.0**（versionCode 26081700，2026-08-23 自手机实机提取，比 2026-08-18 报告的对象 v1.260813.10 新 4 个版本）
> - 方法：jadx 1.5.6 反编译（6,351 个 Java 文件）+ dex 字符串表（93,501 条）+ APK 结构检查（assets / native 库 / 分包）
> - 性质：纯静态分析，未对运行中的 app 或服务端做任何探测
> - 产物：`apk/`（4 个分包）、`dex/`（3 个 dex）、`jadx-out/`、`dex_strings.txt`

---

## 0. 结论（一句话）

**内容安全分类器不在 app 里。** app 内没有任何安全分类模型、推理引擎或分类代码；分类由服务端完成，app 只负责**渲染服务端下发的分类结果**（横幅 + 求助热线入口）和**把用户的"误判"反馈上报回去**。app 内唯一运行的本地 ML 是 Google ML Kit 的**语言识别**（且推理本身在 Google Play Services 进程里，不在 app 进程）。

---

## 1. 证据 A：app 内没有可运行分类器的物质基础

### 1.1 无模型文件

`base.apk` 的 `assets/` 共 576 个文件、4.76 MB，逐一核对无任何模型类文件：

| 类别 | 文件 | 说明 |
|---|---|---|
| 字体 | `latinmodern-math.otf` 等 | 数学/拉丁现代字体 |
| 电话库 | `PhoneNumberMetadataProto_*`（~500 个） | Google libphonenumber 数据 |
| 域名库 | `PublicSuffixDatabase.list` | 公共后缀列表 |
| 其他 | `login_overview_frames.bin`（res/raw） | 登录页动画帧 |

**没有 `.tflite` / `.onnx` / `.pb` / `.pt` / tokenizer / vocab 等任何模型或词表文件。**

### 1.2 无推理引擎

`split_config.arm64_v8a.apk` 的 `lib/` 只有 6 个 .so（合计 ~2.7 MB）：

```
libandroidmath.so               (Compose 数学库)
libandroidx.graphics.path.so    (图形路径)
libdatastore_shared_counter.so  (DataStore, Rust)
libsentry-android.so            (崩溃上报 SDK)
libsentry.so                    (崩溃上报 SDK)
libsqliteJni.so                 (SQLDelight 数据库)
```

**没有 `libtensorflowlite_jni.so` / `libonnxruntime.so` / `libexecutorch.so`。** 单个 TF Lite 推理库通常 5–30 MB，任何神经分类器都无法在无引擎情况下运行。

### 1.3 dex 代码层：没有分类器调用代码

全 dex 字符串表 + 反编译源码中，与 "classify" 相关的命中只有三类，均与安全分类无关：

| 命中 | 实际用途 |
|---|---|
| `android.view.textclassifier.*` | Android 系统自带的文本选择智能操作（系统功能，非 app 逻辑） |
| `classifyToolKind` | 分析埋点：把消息中使用的"工具"分类为分析事件（MobileToolKind） |
| `FailureClassification(reason=...)` | **网络错误**分类（失败原因/阶段/类型），用于错误诊断 UI |

对 `classifier / moderation / toxic / jailbreak / nsfw` 的源码级搜索：**无任何 app 自有类匹配**。

### 1.4 "TOXICITY_DETECTION" 字符串的真相

字符串表里出现 `TOXICITY_DETECTION_CREATE/LOAD/INFERENCE/DOWNLOAD_EVENT`、`NLCLASSIFIER_CLIENT_LIBRARY_*` 等，看起来像本地毒性分类器。逐一追踪后确认是**假阳性证据**：

- 定义处：`defpackage/mem.java`、`rem.java` — 这是 Google ML Kit SDK 自带的**遥测事件 ID 枚举**（覆盖 face/barcode/translator/pose/…全部 ML Kit 模块），随 SDK 依赖打包进 app，与 Anthropic 无关。
- 序列化处：`defpackage/c4m.java`、`x3m.java`（ML Kit Firelog 序列化器）。**关键证据**：序列化方法 `a()` 中，`toxicityDetectionCreateEvent / toxicityDetectionLoadEvent / toxicityDetectionInferenceEvent / toxicityDetectionDownloadEvent` 四个字段**全部写死为 `null`**（第 147–150 行），只有 `onDeviceLanguageIdentificationLogEvent` 被赋真实值。即：ML Kit 遥测 schema 虽然含毒性检测事件位，**app 从未上报过毒性检测事件**——从未使用该能力。

### 1.5 无 Google SafetyCore 集成

字符串表与源码中无 `safetycore` / `content_protection` / 敏感内容警告相关引用（Android 15+ 的本地敏感图片检测走的是 Google 系统组件，与 app 无关）。唯一的 "sensitive" 命中是：
- `SensitiveCleanupState` / `organizations/{uuid}/melange/sensitive_cleanup` — **记忆功能**的敏感数据清理，服务端 API；
- `accessibilityDataSensitive` — 埋点字段脱敏标记。

---

## 2. 证据 B：app 内唯一运行的本地 ML = 语言识别

### 2.1 实现路径（thin client，推理不在 app 进程）

`defpackage/tsl.java` 是 app 的 ML Kit 语言识别包装器。其注册的 Play Services 模块列表（第 13 行）：

```java
String[] {"com.google.android.gms.mlkit.langid",
          "com.google.android.gms.mlkit.nlclassifier",
          "com.google.android.gms.tflite_dynamite"}
```

三个模块都只是**注册名**。实际调用通过 AIDL（`Parcel.writeString` → `jlmVar.m(parcel, 3)`）走 **Google Play Services 进程**，只调用 langid 的检测方法。TFLite Dynamite 模块的推理发生在 GMS 进程内，app 的 APK 和 app 进程都不承载模型与推理。

### 2.2 用途

`defpackage/f9m.java`（包装层）：

```java
strA = f9mVar.a(str.substring(0, Math.min(str.length(), 200)));  // 只取前 200 字符
```

- 输入：文本前 **200 字符**（截断后检测，明显是轻量用途）
- 输出：ISO 639-1 语言代码；空文本返回 `"und"`；过滤 "unknown" 结果；希伯来语 `"iw"` → `"he"` 归一化
- 结论：用于给文本打语言标签的辅助功能（不是安全分类）。

---

## 3. 证据 C：真正的分类链路全部在服务端

app 从服务端**接收**三类分类结果，逐一有 protobuf / API 证据：

### 3.1 流内安全横幅：SafetyBanner（对话消息流）

`com/anthropic/bard/api/v1alpha/ContentBlock`（消息内容块 proto，Wire 编解码）含**重复字段**：

```java
@WireField(adapter = "SafetyBanner#ADAPTER", jsonName = "safetyBanners",
           label = REPEATED, tag = 18)   // ContentBlock.java:42
private final List<SafetyBanner> safety_banners;
```

`SafetyBanner` 结构 = `flag`（String，**redacted = true**，日志中打印为 `flag=██`）+ `helpline`（Helpline：id/名称/电话/短信号码/网页聊天 URL——由服务端配置）。即：**服务端在消息流里附带"该内容触发了某安全标记 + 对应求助资源"，app 原样渲染。**

### 3.2 流内消息标记：FlagBlock / FlagDelta

消息流事件类型 `FlagBlock` / `FlagDelta`（`api/chat/messages/`），携带 `MessageFlag` + `ApiHelpline` + 起止时间戳。`MessageFlag` 的反序列化器（`defpackage/tkc.java`）揭示全部已知标记类型：

```java
strP.equals("self_harm_risk")  → SelfHarmRisk   // h.java
strP.equals("election")        → ElectionFlag   // g.java
其他任意字符串                   → 通用 skc 类型（开放集合，服务端可新增类型而 app 无需发版）
```

app 侧行为：收到标记 → 渲染 `SelfHarmBanner`（自残风险横幅：立即呼叫/短信/在线聊天/查找资源/热线转接），配套埋点 `SelfHarmBannerEvents.*`，"查找资源"跳转 `support.claude.com/articles/8106465` 等帮助文章。**横幅的触发判定完全依赖服务端 flag，app 本地不做任何内容判定。**

### 3.3 Cowork 任务安全标记：独立 REST 端点

```
GET api/organizations/{organizationId}/cowork/messages/{messageId}/safety_flags
→ CoworkSafetyFlagsResponse { flags: [CoworkSafetyFlag(type, helpline)] }
```

标记类型含 `CHILD_SAFETY`、`SUICIDE_AND_SELF_HARM`（与 3.1/3.2 呼应）。同样：**服务端产出，app 拉取后显示。**

### 3.4 用户反馈闭环（服务端分类器的校准回路）

| 常量 | 值 | 含义 |
|---|---|---|
| `ChatFeedbackType.FLAG_SAFETY_CLASSIFIER_FALSE_POSITIVE` | `"sc/false_positive"` | 用户对标记内容点"这是误判"→ 上报服务端 |
| `ChatFeedbackType.FEEDBACK_TYPE_FLAG_HARMFUL` | — | 用户反馈"内容有害"→ 上报服务端 |
| `ChatSnapshotReportCategory.CHILD_SAFETY` 等 11 类 | — | 对话快照举报类别（版权/商标/儿童安全/欺诈/成人内容/暴力仇恨/违禁品/法院命令/私密信息/其他违法/其他） |

`x63.java:160` 证实：用户选择 `FLAG_SAFETY_CLASSIFIER_FALSE_POSITIVE` 反馈时走独立的反馈提交路径——这是服务端分类器"被纠错"的证据：**分类器输出会因用户反馈而调整，调整只能发生在服务端。**

---

## 4. 数据流总图

```
用户输入 ──▶ app（本地仅做语言识别：前200字符 → 语言标签）
                │
                ▼ 加密上传
         Claude 服务端
                │  ★ 内容安全分类在此发生（模型/规则均不可见）
                ├──▶ ContentBlock.safety_banners (tag 18) ──▶ app 渲染安全横幅
                ├──▶ FlagBlock/FlagDelta (self_harm_risk / election) ──▶ app 渲染标记横幅+热线
                └──▶ GET /cowork/.../safety_flags ──▶ app 渲染 Cowork 安全标记
                ▲
app ── 用户反馈/举报 ──▶ 服务端（sc/false_positive、FLAG_HARMFUL、快照举报 11 类）
```

---

## 5. 隐私与数据安全附注（本次分析顺带发现）

1. **正面**：`SafetyBanner.flag` 字段 `redacted = true`（Wire 的 `redact()` 会清空该字段，`toString()` 显示 `██`）——敏感标记值在日志/序列化副本中被主动脱敏。
2. **正面**：本地语言识别只取文本**前 200 字符**，且为 thin client（模型与推理在 GMS，app 不持有模型）。
3. **注意**：`MessageFlag` 与 `SafetyBanner.flag` 是开放字符串集合——服务端可在不发版的情况下新增任意内容标记类型并推送到 app，用户对"新增了哪些分类"没有客户端层面的可见性。
4. **注意**：helpline 数据（热线电话/短信/聊天 URL）由服务端动态配置下发，app 无条件渲染——这是一条可被服务端配置影响 UI 行为的通道。
5. 崩溃上报 SDK（Sentry）以原生库形式打包（libsentry.so）——崩溃日志采集为独立通道，不在本次范围。

---

## 6. 结论回答与"改 APK 能否绕过"的推论

**分类器不在 app 里**，因此：

- 修改/重打包 APK **无法移除或绕过内容安全分类**——分类在服务端消息流中产生（SafetyBanner / FlagBlock / safety_flags），重打包最多只能隐藏横幅 UI，分类与潜在的服务端处置（标记、限制、封禁关联）依旧发生。
- app 内可被改动的只有**展示层**（横幅、热线入口）与**反馈上报**——这不会影响服务端对账号/内容的判定。
- 设备上唯一本地 ML（语言识别）与安全无关，绕过它不产生安全意义。

---

## 7. 证据文件索引

| 证据 | 位置 |
|---|---|
| SafetyBanner proto（含 redacted、helpline） | `jadx-out/sources/com/anthropic/bard/api/v1alpha/SafetyBanner.java` |
| ContentBlock.safety_banners（tag 18） | `jadx-out/sources/com/anthropic/bard/api/v1alpha/ContentBlock.java:42` |
| MessageFlag 反序列化（flag 类型清单） | `jadx-out/sources/defpackage/tkc.java:14-17` |
| FlagBlock（消息流标记块） | `jadx-out/sources/com/anthropic/claude/api/chat/messages/FlagBlock.java` |
| SelfHarmRisk 标记 | `jadx-out/sources/com/anthropic/claude/api/chat/messages/h.java` |
| CoworkSafetyFlag / Response | `jadx-out/sources/com/anthropic/claude/sessions/types/CoworkSafetyFlag.java`、`CoworkSafetyFlagsResponse.java` |
| ML Kit 遥测枚举（TOXICITY_* 来源） | `jadx-out/sources/defpackage/mem.java`、`rem.java` |
| ML Kit 序列化器（toxicity 字段全 null 铁证） | `jadx-out/sources/defpackage/c4m.java:147-150`、`x3m.java:147-150` |
| 语言识别包装（模块注册 + AIDL 调用） | `jadx-out/sources/defpackage/tsl.java:13` |
| 语言识别使用（前 200 字符） | `jadx-out/sources/defpackage/f9m.java:19-50`、`l3m.java:25` |
| 反馈类型 sc/false_positive | `jadx-out/sources/com/anthropic/claude/api/chat/ChatFeedbackType.java:29` |
| 举报类别 CHILD_SAFETY 等 | `jadx-out/sources/com/anthropic/claude/api/share/ChatSnapshotReportCategory.java` |
| 帮助文章链接（含 REAL_TIME_SAFEGUARDS） | `jadx-out/sources/defpackage/myi.java` |
| 原始 APK 分包 | `apk/`（base + arm64_v8a + xxxhdpi + zh） |
| 全量字符串表 | `dex_strings.txt`（93,501 条） |
