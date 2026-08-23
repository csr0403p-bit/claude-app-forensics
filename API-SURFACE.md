# Claude Android App API 面全映射报告

> 问题：**app 与哪些服务端端点通信？每个请求带什么认证、什么数据？**
>
> - 对象：`com.anthropic.claude` v1.260817.0（versionCode 26081700，2026-08-23 自手机实机提取）
> - 方法：jadx 1.5.6 反编译（6,351 个 Java 文件）+ dex 字符串表（93,501 条）。端点取自 Wire RPC 接口注解（`@rsd/@osd/@mu8/@t76/@n99`，对应 PUT/POST/GET/DELETE/自定义方法，推断依据：参数形态——`@a0e`=路径参数、`@jcf`=查询参数、`@z02`=请求体、`@mqi`=SSE 流式、`@pxc`=multipart 上传、`@xb9`=附加请求头）
> - 性质：纯静态分析；未向任何服务端发送请求
> - 端点总数：**约 180 个唯一路径**
> - 关联报告：`CLASSIFIER-ANALYSIS.md`（分类器定位）、`README.md`（2026-08-18 设备信任与数据采集）

---

## 0. 总览

| 项 | 值 |
|---|---|
| 主 API 主机 | `api.claude.ai`（另有 `api.anthropic.com`、内部环境 `api.claude-ai.staging.ant.dev`） |
| 传输 | HTTPS + OkHttp（Cronet 传输层）；SSE 流式（`Accept: text/event-stream`）；WebSocket（`wss://`，路径 `ws/voice/organizations/{org}/chat_conversations/{chat}`、`/api/ws/organizations/`） |
| 认证体系 | 会话令牌（Authorization）+ `X-Organization-Uuid` + `Anthropic-Version: 2023-06-01` + `Anthropic-Beta: ccr-byoc-2025-07-29` + `X-Trusted-Device-Token`（受信设备，见 2026-08-18 报告通道②） |
| RPC 风格 | 自定义 Wire RPC 注解 + kotlinx.serialization JSON 请求体；SSE 返回流式 JSON 事件 |
| 遥测 | Segment（api.segment.io）、Sift（api3.siftscience.com）、Sentry（崩溃）、Datadog RUM（`/api/v2/rum`）、OpenTelemetry spans（`/api/v2/spans`）、自有事件端点 `event_logging/v2/batch`、Firebase FCM |

**一句话**：app 的所有对话、记忆、项目、Cowork、代码功能都走 `api.claude.ai` 的 `organizations/{org}/...` REST 面；认证头把**组织、API 版本、受信设备令牌**挂在每个请求上；另有 4 条独立的遥测通道把**设备信息**发往 Anthropic 与第三方。

---

## 1. 主机清单

| 主机 | 用途 | 归属 |
|---|---|---|
| `api.claude.ai` | 主 API（全部业务端点） | Anthropic |
| `api.anthropic.com` | Console/API 密钥面（api-key 相关字符串） | Anthropic |
| `api.claude-ai.staging.ant.dev` / `claude-ai.staging.ant.dev` | 内部 staging 环境（代码含 `api_force_staging` 开关） | Anthropic |
| `assets.claude.ai` | 静态资源 CDN | Anthropic |
| `claude.findahelpline.com` | 自残风险横幅的求助热线查找（helpline 数据源） | 第三方服务 |
| `support.claude.com` | 帮助文章（横幅"浏览资源"目标） | Anthropic |
| `api3.siftscience.com` | Sift 设备指纹（v3 API，见 8/18 报告通道③） | 第三方 |
| `api.segment.io/v1` | Segment 分析事件上报 | 第三方 |
| Sentry（io.sentry DSN，部署于 app 内） | 崩溃/性能上报 | 第三方 |
| `fcmregistrations.googleapis.com` / `firebaseinstallations.googleapis.com` | Firebase 推送注册（`Goog-Firebase-Installations-Auth` 头） | Google |
| `accounts.google.com` | Google 登录 | Google |
| `play.google.com` | Play Integrity 完整性证明 | Google |
| 本地 | `http://10.0.2.2`（模拟器调试）、`officeproxy.local`/`pdfproxy.local`（Office/PDF 本地渲染代理） | 本机 |

---

## 2. 认证与设备标识头

### 2.1 主 API 拦截器（`defpackage/jmh.java`，每个请求都挂）

```java
sxfVarB.a("X-Organization-Uuid", this.u);          // 组织 UUID
sxfVarB.a("Anthropic-Version", "2023-06-01");       // API 版本
sxfVarB.a("Anthropic-Beta", "ccr-byoc-2025-07-29"); // CCR 受信设备 beta（8/18 报告通道②）
sxfVarB.a("X-Trusted-Device-Token", strJ);          // 受信设备令牌（安全存储读出）
```

另有 `AUTHORIZATION`（会话令牌）与 `cf-aig-authorization`（Cloudflare 边缘认证）头字符串存在；`defpackage/q2c.java` 是一份**敏感头黑名单**（authorization、x-api-key、cf-aig-authorization、各种云厂商 key 头共 60+ 个）——用于请求日志脱敏，说明日志层会过滤这些头的值。

### 2.2 受信设备认证端点（与 8/18 报告通道②呼应）

```
auth/trusted_devices                          POST  登记设备公钥
auth/trusted_devices/{device_id}/rotate_reattest  POST  轮换重认证
auth/session_reattest/device_key              GET/POST 会话重认证取密钥
auth/session_reattest/device_key/challenge    POST  重认证挑战
```

### 2.3 登录/登出/SSO

```
auth/send_magic_link          POST  发送魔法链接（邮箱）
auth/verify_magic_link        POST  验证魔法链接
auth/send_phone_code          POST  发送手机验证码
auth/verify_phone_code        POST  验证手机验证码
auth/verify_google_mobile     POST  Google 移动端登录验证
auth/logout                   POST  登出
enterprise_auth/sso_callback  GET   企业 SSO 回调
```

---

## 3. 端点清单（按功能域分组，共 ~180 个）

> 动词标注：G=GET、P=POST、U=PUT、D=DELETE、S=SSE 流式。动词为依据注解形态的推断（@mu8=G、@osd=P、@rsd=U、@t76=D），个别以服务端实现为准。

### 3.1 启动与账户（bootstrap / account / profile）

```
bootstrap                                    GET   全局配置/实验开关/横幅
bootstrap/{organizationUuid}/app_start       POST  应用启动上报（每次冷启动）
account                                      GET   账户信息
account/settings                             GET   账户设置
account/accept_legal_docs                    POST  接受法律条款
account/deletion-allowed                     GET   是否允许销号
account/grove_notice_viewed                  POST  公告已读
account/send_product_email_link              POST  发送产品邮件链接
api/v2/profile                               GET   个人资料
legal                                        GET   法律文案
```

### 3.2 隐私同意（privacy consents）

```
v1/privacy-consents                          P    记录同意状态
v1/privacy-consents/check                    P    查询同意状态
accounts/me/consents                         GET   同意列表
accounts/me/consents/check                   POST  同意检查
accounts/me/consents/revoke                  POST  撤回同意
```

### 3.3 对话（chat_conversations —— 主业务面）

```
organizations/{organization}/chat_conversations                      G   列表（query: consistency/searchQuery/limit/offset/starred）
organizations/{organization}/chat_conversations                      P   新建
organizations/{organization}/chat_conversations/{chat}               G   读取（query: rendering_mode/render_all_mobile_tools/tools/return_dangling_human_message/include_extracted_content）
organizations/{organization}/chat_conversations/{chat}               U   更新
organizations/{organization}/chat_conversations/{chat}               D   删除
organizations/{organization}/chat_conversations/delete_many          P   批量删除
organizations/{organization}/chat_conversations/move_many            P   批量移动（到项目）
organizations/{organization}/chat_conversations/{chat}/completion    S   流式补全（SSE）
organizations/{organization}/chat_conversations/{chat}/completion2   S   流式补全 v2（SSE）
organizations/{organization}/chat_conversations/{chat}/retry_completion  S   重试补全（SSE）
organizations/{organization}/chat_conversations/{chat}/retry_completion2 S   重试补全 v2（SSE）
organizations/{organization}/chat_conversations/{chat}/stop_response P   停止生成
organizations/{organization}/chat_conversations/{chat}/title         P   生成标题
organizations/{organization}/chat_conversations/{chat}/tool_result   P   上报工具执行结果
organizations/{organization}/chat_conversations/{chat}/tool_approval P   工具审批
organizations/{organization}/chat_conversations/{chat}/model_fallback U   模型回退
organizations/{organization}/chat_conversations/{chat}/share         P   分享
organizations/{organization}/chat_conversations/{chat}/shares        G   分享列表
organizations/{organization}/chat_conversations/{chat}/chat_messages/{message}/chat_feedback  P/U  消息反馈
organizations/{organization_uuid}/chat_conversations/{chat}/chat_messages/{message}/flags     D   删除消息标记（带 body）
organizations/{organization_uuid}/chat_conversations/{chat}/task/{task_id}/stop               P   停止任务
organizations/{organization_uuid}/chat_conversations/{chat}/task/{task_id}/mobile_status      G   任务状态
organizations/{organization_uuid}/chat_conversations/{chat}/composer_notices                  G   输入框通知
organizations/{organization_uuid}/chat_conversations/{chat}/composer_notices/action           P   输入框通知动作
organizations/{organization_uuid}/conversation/search/v2                                     G   对话搜索（query: query/n/page_token）
organizations/{organization}/convert_document                          P   文档转换（multipart）
organizations/{organization}/upload / {organization}/upload             P   附件上传（multipart）
organizations/{organization_uuid}/conversations/{conversation}/files/prepare-upload           P   文件上传预检
organizations/{organization_uuid}/conversations/{conversation}/wiggle/upload-file             P   上传文件
organizations/{organization_uuid}/conversations/{conversation}/wiggle/convert-file-to-artifact P  文件转 artifact
organizations/{organization_uuid}/conversations/{conversation}/wiggle/delete-file             P   删除文件
{organization}/files/{fileUuid}/preview                                 G   文件预览
```

### 3.4 记忆（melange / memory）

```
organizations/{organization_uuid}/memory                    G   记忆列表
organizations/{organization_uuid}/memory/settings           G/U 记忆设置
organizations/{organization_uuid}/melange/list              G   记忆条目列表
organizations/{organization_uuid}/melange/read              G   读取记忆
organizations/{organization_uuid}/melange/edit              P   编辑记忆
organizations/{organization_uuid}/melange/delete            P   删除记忆
organizations/{organization_uuid}/melange/sensitive_cleanup P   敏感记忆清理
organizations/{organization_uuid}/melange/write_status/{chat} G  记忆写入状态
organizations/{organization_uuid}/reflections/settings      G   反思设置
organizations/{organization_uuid}/reflections/time_spent    G   使用时长统计
```

### 3.5 Cowork（自主任务）

```
api/organizations/{organizationId}/cowork/sessions                        P   创建任务会话
api/organizations/{organizationId}/cowork/sessions/{sessionId}/feedback   P   会话反馈
api/organizations/{organizationId}/cowork/messages/{messageId}/safety_flags G  消息安全标记（见分类器报告 3.3）
api/organizations/{organizationId}/cowork/approval_hints                  G   审批提示
api/organizations/{organizationId}/cowork/remote_devices                  G   远程设备
api/organizations/{organizationId}/cowork/scheduled_tasks                 G/P 定时任务
api/organizations/{organizationId}/cowork/scheduled_tasks/{triggerId}     G/U/D 单个定时任务
api/organizations/{organizationId}/cowork/scheduled_tasks/{triggerId}/run P   手动运行任务
organizations/{organization}/cowork_settings                              G/U Cowork 设置
organizations/{organization}/remote_action                               P   远程动作
organizations/{organization_uuid}/skills/list-skills                      G   技能列表
```

### 3.6 项目与文件（projects / artifacts）

```
organizations/{organization_uuid}/projects                                 G/P 项目列表/新建
organizations/{organization_uuid}/projects_v2                              G   项目 v2
organizations/{organization_uuid}/projects/{project_uuid}                  G/U/D 单个项目
organizations/{organization_uuid}/projects/{project_uuid}/conversations    G   项目对话
organizations/{organization_uuid}/projects/{project_uuid}/docs             G/P 项目文档
organizations/{organization_uuid}/projects/{project_uuid}/docs/{doc_uuid}  G/U/D 单个文档
organizations/{organization_uuid}/projects/{project_uuid}/files            G   项目文件
organizations/{organization_uuid}/projects/{project_uuid}/upload           P   项目上传
organizations/{organization_uuid}/projects/{project_uuid}/files/delete_many P  批量删文件
organizations/{organization_uuid}/projects/{project_uuid}/kb/resync        P   知识库重同步
organizations/{organization_uuid}/projects/{project_uuid}/kb/stats         G   知识库统计
organizations/{organization_uuid}/projects/{project_uuid}/recents          G   最近使用
api/organizations/{organization_uuid}/projects/{project_uuid}/files        G   项目文件
v1/filestore/fs/createFile                                                P   文件存储创建
organizations/{organization}/publish_artifact                              P   发布 artifact
organizations/{organization}/published_artifacts/{artifactId}              G   已发布 artifact
organizations/{organization}/user_artifacts                                G   用户 artifact 列表
organizations/{organization}/artifact-versions/{artifactId}/visibility     P   artifact 可见性
organizations/{organization}/artifacts/{conversationUuid}/versions         G   artifact 版本
published_artifacts/{artifactId}                                           G   公开 artifact
```

### 3.7 分享与举报（share / snapshots）

```
v1/sessions-share                              P   分享会话
v1/sessions-share/{shareId}                    G   读取分享
organizations/{organization}/share/{snapshot}  G   分享快照
organizations/{organization}/shares            G   分享列表
organizations/{organization}/chat_snapshots/{snapshot}          G   对话快照
organizations/{organization}/chat_snapshots/{snapshot}/report   P   举报快照（11 类：CHILD_SAFETY 等）
organizations/{organization}/app_feedback                       P   应用反馈
```

### 3.8 MCP（模型上下文协议）

```
organizations/{organization}/mcp/remote_servers        G   远程 MCP 服务器
organizations/{organization}/mcp/v2/bootstrap          G   MCP 引导
organizations/{organization}/mcp/probe                 P   探测
organizations/{organization}/mcp/attach_prompt         P   附加提示词
organizations/{organization}/mcp/attach_resource       P   附加资源
organizations/{organization}/mcp/start-auth/{server}   P   开始认证
organizations/{organization}/mcp/logout/{server}       P   登出服务器
api/mcp/auth_callback / mcp/auth_callback              G   OAuth 回调
v1/toolbox/shttp/mcp/                                  P   MCP shttp 通道
```

### 3.9 代码（Code —— Claude Code 移动端）

```
v1/code/sessions                                      G/P 代码会话
v1/code/sessions/watch                                S   会话监视（SSE）
v1/code/sessions/{sessionId}                          G/U/D 单个会话
v1/code/sessions/{sessionId}/events                   G   事件
v1/code/sessions/{sessionId}/events/stream            S   事件流（SSE）
v1/code/sessions/{sessionId}/ping                     P   心跳
v1/code/sessions/{sessionId}/archive / unarchive      P   归档/取消归档
v1/code/sessions/{sessionId}/mark_read                P   标记已读
v1/code/sessions/{sessionId}/mcp-servers              G   MCP 服务器
v1/code/sessions/{sessionId}/client/presence          P   客户端在线状态
v1/code/sessions/{sessionId}/move-to-cloud            P   转到云端
v1/code/sessions/{sessionId}/share                    P   分享
v1/code/shared-sessions/{shareId}                     G   读取分享
v1/code/channels                                      G   频道
v1/code/channels/{channel_id}                         G   频道详情
v1/code/channels/{channel_id}/messages                G/P 频道消息
v1/code/channels/{channel_id}/messages/stream         S   消息流（SSE）
v1/code/channels/{channel_id}/threads/{thread_id}     G   线程
v1/code/channels/{channel_id}/timeline                G   时间线
v1/code/channels/{channel_id}/config                  G   配置
v1/code/channels/{channel_id}/mark_read               P   已读
v1/code/channels/{channel_id}/client/presence         P   在线状态
v1/code/triggers                                      G/P 触发器
v1/code/triggers/{triggerId}                          G/U/D 单个触发器
v1/code/runners/self-hosted/pools                     G   自托管运行池
v1/code/github/batch-branch-status                    P   批量分支状态
v1/code/github/compare-refs                           P   比较引用
v1/code/github/get-file-content                       P   取文件内容
v1/code/github/set-pr-auto-merge                      P   设置 PR 自动合并
v1/code/github/subscribe-pr / unsubscribe-pr          P   订阅 PR
api/github/organizations/{organizationId}/github/{owner}/{repo}/branches  G   GitHub 分支
api/github/organizations/{organizationId}/github_create_pr                P   创建 PR
api/organizations/{organizationId}/code/repos/all                        G   仓库列表
api/organizations/{organizationId}/code/repos/resync                     P   重同步
api/organizations/{organizationId}/code/shares/scan_secrets              P   扫描密钥
organizations/{organization}/sync/github/auth/start / finish             P   GitHub OAuth
organizations/{organization}/sync/github/auth                            G   状态
v1/environment_providers/private/organizations/{organizationId}/cloud/create       P   云环境创建
v1/environment_providers/private/organizations/{organizationId}/environments         G   环境列表
v1/environment_providers/private/organizations/{organizationId}/environments/{env}   G/U/D 单个环境
v1/environment_providers/private/organizations/{organizationId}/environments/{env}/archive P 归档
v1/sessions / v1/sessions/{sessionId}                 G   通用会话（Code 复用）
v1/sessions/{sessionId}/events                        G   事件
v1/sessions/{sessionId}/share-status                  G   分享状态
```

### 3.10 通知（notification）

```
organizations/{organization_uuid}/notification/channels           G   通知频道
organizations/{organization_uuid}/notification/preferences        G/U 通知偏好
organizations/{organization_uuid}/notification/push/track_open    P   推送打开追踪
organizations/{organization_uuid}/notification/debug/test_push    P   测试推送（调试）
```

### 3.11 计费与订阅（billing / iap）

```
google-play-iap/purchase                          P   Play 内购验证
google-play-iap/transaction                       P  交易
organizations/{orgId}/usage                       G   用量
organizations/{orgId}/overage_spend_limit         G/U 超额支出上限
organizations/{organization_uuid}/prepaid/credits G   预付费积分
organizations/{organization_uuid}/prepaid/iap/android P  Android 内购
organizations/{organization}/paused_subscription_details G  暂停订阅详情
organizations/{organization}/referral/eligibility G   推荐资格
```

### 3.12 体验与横幅（experiences —— 服务端驱动 UI）

```
organizations/{organization}/experiences         G   体验配置
organizations/{organization}/experiences/action  P   体验动作
organizations/{organization}/experiences/track   P   体验追踪
```

### 3.13 遥测与调试（Anthropic 自有）

```
event_logging/v2/batch    P   事件批量上报
api/v2/rum                P   Real User Monitoring（性能遥测）
api/v2/spans              P   OpenTelemetry 追踪 spans
api/frame/                P   帧（前端埋点）
dust/generate_session_title / generate_title_and_branch（api/organizations/{id}/dust/...） P  AI 标题生成
```

---

## 4. 流式与 WebSocket

| 通道 | 路径 | 说明 |
|---|---|---|
| SSE 流式补全 | `.../chat_conversations/{chat}/completion{2}`、`retry_completion{2}` | 请求头强制 `Accept: text/event-stream`；返回 `gi2` 流式包装 |
| SSE 代码事件 | `v1/code/sessions/{id}/events/stream`、`.../messages/stream`、`v1/code/sessions/watch` | Code 功能实时事件 |
| WebSocket | `ws/voice/organizations/{org}/chat_conversations/{chat}` | 语音对话（`wss://`） |
| WebSocket | `/api/ws/organizations/` | 组织级 WS 通道 |
| 语音辅助 | `v2/voice/audio_capture/share_consent`、`v2/voice/audio_capture/delete_all` | 语音音频捕获同意与删除 |

---

## 5. 遥测与第三方上报通道（每条请求都带什么）

| 通道 | 去向 | 携带内容 |
|---|---|---|
| Segment（`api.segment.io/v1`） | 第三方 | 每个事件附 `ant_device_id`（设备 ID）+ `primary_language`（主语言）+ `traits`（用户属性，**排除 email**）（`defpackage/l66.java`） |
| Sift（`api3.siftscience.com`） | 第三方 | 设备指纹（android_id、SIM、IP、root 证据等，见 8/18 报告通道③） |
| Sentry（原生 SDK） | 第三方 | 崩溃、性能、Sentry Replay 会话 |
| Datadog RUM（`/api/v2/rum`） | Anthropic 端点 | RUM 性能事件（`datadog_rum_profiler_sample_rate` 配置字符串存在） |
| `/api/v2/spans` | Anthropic | 追踪 spans |
| `event_logging/v2/batch` | Anthropic | 事件批量上报 |
| Firebase FCM | Google | 推送注册（installation token） |
| Play Integrity | Google | 登录时设备完整性证明（8/18 报告通道①） |

---

## 6. 关键端点请求体（示例：流式补全）

`organizations/{organization}/chat_conversations/{chat}/completion2`（POST, SSE）请求体 `ChatCompletionRequest`，包含：

- 会话上下文（conversation UUID、父消息 UUID）
- 用户提示文本 / 附件（文件 UUID、图片）
- 模型与推理参数（model、thinking 开关、max_tokens、temperature 等）
- 工具定义与工具结果（MCP / 内置工具）
- 设备与客户端能力标记

即：**用户输入、附件元数据、会话上下文、所用工具及结果**随每个补全请求上传（响应为 SSE 流式 JSON，内含 ContentBlock——安全横幅 SafetyBanner 也在此通道下发，见分类器报告）。

---

## 7. 隐私与安全含义小结

1. **组织标识无处不在**：每个请求头带 `X-Organization-Uuid`——账号→组织的绑定在传输层恒定可关联。
2. **受信设备令牌挂在每个请求**：`X-Trusted-Device-Token` 使每个请求可关联到具体设备与硬件密钥（配合 8/18 报告通道②的每消息签名）。
3. **设备指纹至少走 3 条独立通道**：Sift（启动时）、Segment（ant_device_id，每个分析事件）、Play Integrity（登录）。
4. **脱敏机制存在**：`q2c.java` 维护 60+ 敏感头黑名单用于日志脱敏；`SafetyBanner.flag` 与 helpline 字段 redacted（见分类器报告 5.1）。
5. **服务端驱动 UI 面**：bootstrap + experiences + composer_notices 三个通道让服务端可动态下发配置、横幅、输入框通知；MessageFlag 类型为开放集合。
6. **调试端点**：`notification/debug/test_push`、`api_force_staging` 开关存在于生产构建中。

---

## 8. 方法与局限

- 端点路径全部来自反编译的 RPC 接口注解，为静态事实；HTTP 动词为依据注解形态（GET/POST/PUT/DELETE 注解族）与参数形状的推断，个别以服务端实现为准。
- 未做动态抓包，因此：请求体实际值、服务端未在客户端引用的隐藏端点、以及服务端对请求的校验逻辑不在本报告范围内。
- 流式端点（SSE）与 WebSocket 的帧内容格式仅从客户端解析逻辑可见，未列全量事件类型。
