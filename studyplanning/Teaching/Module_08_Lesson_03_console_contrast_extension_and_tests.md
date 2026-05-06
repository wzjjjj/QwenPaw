# Module 08 / Lesson 03：Console 对比 + 新渠道设计清单 + 如何验证

## 本课总览（总）

在 Lesson 02 你已经看到：Weixin 是典型“真实 IM 渠道”，它需要接收侧高可用（长轮询/重试/去重），需要严格串行化同一会话，并且回包强依赖 `meta`（context_token 等）。

本课用 Console 做对比，帮你确认 Channels 的抽象边界：哪些东西是 BaseChannel/ChannelManager 的统一能力，哪些东西应该在每个渠道里做；最后给出“新增一个渠道必须做什么”的可执行清单，并告诉你怎么用现有测试与契约验证。

## 学习目标

- 能解释：为什么 Console 的交互模型与 Weixin 不同（以及它为何仍复用 BaseChannel 的契约）
- 能列出：新增一个渠道的必做/选做项（按接口契约）
- 能知道：自定义渠道如何注册、如何挂自定义 HTTP 路由
- 能按仓库现有结构写出最小验证路径（单测/契约/doctor）

## 先修知识

- 已读 Lesson 01（异步心智模型）与 Lesson 02（weixin 端到端链路）

## 本节要回答的关键问题

1. Console 的 `stream_one` 为何看起来像“自己消费”，而不是“入队→消费”？
2. BaseChannel 的“必做接口”有哪些？哪些是强烈建议覆盖的？
3. 新增自定义渠道如何被发现？如果需要 webhook endpoints 应该怎么挂？
4. 如何快速验证一个新渠道：能收、能发、不会把系统拖死？

## 核心概念（分）

### 1) Console vs Weixin：模型差异与抽象边界

Weixin 的约束来自平台：

- 接收发生在后台线程（长轮询），不能阻塞主 loop
- 收到消息后需要入队，主 loop 串行消费，再回包
- 回包依赖 meta（context_token 等）

Console 的约束来自本地交互：

- 输入通常来自本地 UI/CLI，天然已经在主 loop 或可控线程内
- 输出不是“调用平台 API”，而是“打印/推送到本地 store”
- 体验上更像“把 runner 的事件流直接展示出来”

Console 的 `stream_one` 路线：

- 调用方：ConsoleChannel 的 consume 路径（或外部控制台交互）
- 被调：`ConsoleChannel.stream_one(payload)`，内部会把 payload → request，然后 `async for event in self._process(request)`，把消息渲染打印  
  [ConsoleChannel.stream_one](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/console/channel.py#L332-L439)

这并不意味着 Console “不需要 manager queue”，而是它选择了更贴合本地展示的消费方式。真正的抽象边界仍然成立：

- 入站最终都要变成 `AgentRequest`
- 出站最终都要把 runner 产出的 message event 变成“某种发送”

### 2) 新增一个 Channel：必须做什么（按契约分层）

这一节就是你要的“设计清单”。你可以把它当作写渠道的验收标准。

#### A) 类与注册（必做）

- 实现 `BaseChannel` 子类，并设置类属性 `channel`（唯一 key）
  - Base 定义：`BaseChannel.channel`  
    [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L78-L87)
- 让系统能找到你的类（两种方式）：
  - **内置渠道**：加入 built-in registry（修改 `_BUILTIN_SPECS`）  
    [registry.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/registry.py#L20-L37)
  - **自定义渠道（推荐）**：放到 `CUSTOM_CHANNELS_DIR`，系统通过 `_discover_custom_channels()` 扫描所有 `BaseChannel` 子类并用 `obj.channel` 注册  
    [registry.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/registry.py#L98-L130)

#### B) 生命周期（必做）

- `async def start(self) -> None`
  - 职责：启动接收（webhook server 的注册、WS 连接、long-poll 线程等）
  - 要求：不要阻塞主 loop；如果要阻塞，放到线程/后台 task
- `async def stop(self) -> None`
  - 职责：取消后台 task、停止线程/loop、关闭连接、释放文件句柄

Base 抽象位置：  
[base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L1272-L1277)

#### C) 入站适配：平台消息 → AgentRequest（必做）

- `def build_agent_request_from_native(self, native_payload: Any) -> AgentRequest`
  - 把平台 payload 解析成 runtime `content_parts`（TextContent/ImageContent/...）
  - 计算或提取 `session_id`（决定同会话串行）
  - 调 `build_agent_request_from_user_content(...)` 生成 `AgentRequest`
  - 必须确保回包所需字段能沿链路传下去：通常放在 `meta`，并设置到 `request.channel_meta`（Base/子类都可能会做 attach）

Base 的辅助构造方法：  
[build_agent_request_from_user_content](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L607-L640)

Weixin 实现示例：  
[WeixinChannel.build_agent_request_from_native](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L251-L273)

Console 实现示例：  
[ConsoleChannel.build_agent_request_from_native](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/console/channel.py#L255-L276)

#### D) 出站发送：Agent 输出 → 平台 API（必做）

最小可用：

- `async def send(self, to_handle: str, text: str, meta: Optional[Dict[str,Any]] = None) -> None`
  - Base 默认 `send_content_parts` 最终会调用它  
    [BaseChannel.send](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L1278-L1288)

强烈建议（现实 IM 基本都要）：

- 覆盖 `async def send_content_parts(self, to_handle: str, parts: List[OutgoingContentPart], meta: Optional[Dict[str,Any]] = None) -> None`
  - 才能原生发送图片/文件/视频等（否则默认会退化为把链接拼进文本）  
    - Base 默认行为：[BaseChannel.send_content_parts](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L1129-L1181)
    - Weixin 覆盖示例：[WeixinChannel.send_content_parts](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L1066-L1164)

#### E) 队列模式（必做决策）

默认建议：`uses_manager_queue = True`（Base 默认值）  
[base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L83-L87)

这意味着：

- ChannelManager 会为该渠道创建“按 session 串行”的消费队列
- 渠道接收侧只需要调用 `self._enqueue(payload)`，不要直接跑 runner

只有在极少数场景（例如 Console 这种本地打印交互、或你明确要自己实现队列）才考虑绕开 manager queue，但这会提高复杂度（你要自己保证串行、合并、错误处理等）。

#### F) 会话与路由（强烈建议）

你需要想清楚三件事：

1. **session_id 如何定义**（私聊 vs 群聊隔离、同一用户多个会话隔离）
2. **to_handle 是什么**（是 user_id？session_id？chat_id？）
3. **主动推送如何定位目标**（Cron/Heartbeat 会调用 to_handle_from_target）

相关接口点：

- `resolve_session_id(sender_id, channel_meta)`（建议覆盖）  
  [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L595-L606)
- `get_to_handle_from_request(request)`（决定回包目标）  
  [weixin/channel.py 实现](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L240-L244)
- `to_handle_from_target(user_id, session_id)`（Cron/Heartbeat 的 dispatch 目标）  
  [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L1289-L1297)

#### G) 自定义 HTTP 入口（选做，但 webhook 渠道常用）

如果你的渠道需要额外 HTTP endpoints（例如 webhook 回调、扫码登录页）：

- 在自定义渠道模块里提供 `register_app_routes(app)`，系统会在 app 启动时调用  
  [register_custom_channel_routes](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/registry.py#L136-L189)
- 路由必须挂在 `/api/` 前缀下，否则会被 SPA catch-all 吞掉（仓库里有显式警告）  
  [registry.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/registry.py#L149-L186)
- 调用点在 FastAPI app 装配时：  
  [ _app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L645-L646)

#### H) 可观测性与自检（建议）

- `health_check()`：让控制台/运维知道渠道是否健康  
  Base 默认实现：[base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L1255-L1271)
- `doctor_connectivity_notes(...)`：`copaw doctor --deep` 的可达性提示（比如依赖域名、端口）  
  Base 默认实现：[base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L88-L111)

### 3) 如何验证一个新渠道（尽量复用现有测试）

仓库里渠道测试入口在：`tests/unit/channels/`，并且有 contract tests。

建议最小验证顺序（从低成本到高成本）：

1. 只验证“构造/启动不崩”：能被 registry 发现、能被 ChannelManager.from_config 初始化  
   - 参考 `ChannelManager.from_config` 如何按签名过滤 kwargs、如何处理 enabled  
     [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L113-L208)
2. 验证“入站契约”：给 `build_agent_request_from_native` 一个 native payload，能产出 AgentRequest（session_id/user_id/content 都正确）
3. 验证“出站契约”：给 `send_content_parts/send` 一个最小 parts，能发送（可以先用 mock client）
4. 验证“串行与合并”：同 session 连续入队两条，确保不会并发跑两个 runner（看日志/计数器）

## 代码走读路线（分）

1. Console 的 stream 模式：  
   [ConsoleChannel.stream_one](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/console/channel.py#L332-L439)
2. BaseChannel 的统一消费入口（你新增渠道时最终都要走到这里）：  
   [BaseChannel._consume_one_request](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L797-L863)
3. ChannelManager 的队列化与串行消费：  
   [ChannelManager._enqueue_one](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L258-L304)、
   [ChannelManager._consume_queue](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L365-L461)
4. 自定义渠道如何被发现与加路由：  
   [channels/registry.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/registry.py#L98-L189)

## 动手练习（可执行）

练习 1：为你的“目标渠道”（例如你们的快平）写出一份“渠道契约草图”。

- 输入（inbound）：你能从平台拿到哪些字段？你准备怎么生成：
  - `sender_id`
  - `session_id`
  - `content_parts`
  - `meta`（列出回包必须字段）
- 输出（outbound）：平台回包需要哪些字段？你准备把它们放进 `meta` 的哪些 key？
- 并发策略：群聊/私聊怎么隔离？同一个 session 是否允许并发？

练习 2：读 weixin 的 `meta` 字段，思考你自己的平台里等价的最小集合是什么：  
[WeixinChannel._on_message](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L707-L716)

## 验收清单

- [ ] 能解释 Console 为什么更像“展示 runner 事件流”，但仍遵守入站/出站契约
- [ ] 能列出新增渠道的必做项：start/stop、build_agent_request_from_native、send
- [ ] 能解释为什么建议覆盖 send_content_parts
- [ ] 能说清自定义渠道的注册方式与 register_app_routes 的 /api/ 约束
- [ ] 能写出你自己的渠道“native payload 最小字段清单”

## 报告素材提炼（用于你后续写 Channel 实现报告）

这一节把 Channels 的“必须讲清楚的核心点”提炼成可直接复用到报告里的提纲。你可以按这个结构去写：先讲抽象，再讲链路，再讲并发与边界，最后讲扩展与验证。

### 1) 一句话定义 Channels

Channels = “平台消息适配层 + 统一队列与串行化 + runner 事件流渲染发送”。  
平台侧只关心收发；agent 侧只关心 `AgentRequest` 与 `Event`；中间用 `ChannelManager + BaseChannel` 解耦。

### 2) 关键抽象与职责边界

- **BaseChannel（平台适配器）**
  - 入站：平台 payload → runtime `content_parts` → `AgentRequest`  
    [BaseChannel.build_agent_request_from_native](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L642-L656)
  - 出站：消费 runner 的 event 流 → 渲染/过滤 → 发送到平台  
    [BaseChannel._run_process_loop](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L865-L914)
- **ChannelManager（队列与调度器）**
  - 管理：启用哪些渠道、start/stop 生命周期、把 enqueue callback 注入给 channel  
    [ChannelManager.start_all](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L462-L493)
  - 路由：提取 QueueKey `(channel_id, session_id, priority)` 并入队  
    [_enqueue_one](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L258-L304)
  - 消费：每个 QueueKey 一条 consumer 协程，串行 + batch merge  
    [ChannelManager._consume_queue](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L365-L461)
- **UnifiedQueueManager（按 key 动态建队列/consumer）**
  - QueueKey = `(channel_id, session_id, priority_level)`；首次消息到达时按需创建 queue + consumer task  
    [UnifiedQueueManager.enqueue](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/unified_queue_manager.py#L119-L213)
- **runner/process（触发 agent 的统一入口）**
  - channel 不持有“自己的 runner”，它只持有一个 `process` 函数引用，默认就是 `runner.stream_query`  
    [make_process_from_runner](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/utils.py#L141-L153)
  - `ProcessHandler` 的核心语义：给我 request，我给你 event 流  
    [ProcessHandler](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L63-L66)

### 3) 端到端链路（从“收到消息”到“触发 agent”）

你在报告里至少要把这条链路讲清楚（读者只要跟着走一遍就懂系统）：

- 接收侧（channel 接入代码）：收到平台消息 → 组装 native payload → 调用 `self._enqueue(payload)`  
- 主 loop（ChannelManager）：`enqueue → _enqueue_one` 提取 `(channel_id, session_id, priority)` 并交给统一队列  
  [ChannelManager.enqueue](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L352-L363)
- consumer（串行消费同一 key）：`_consume_queue` 拿 payload → drain 组成 batch → `_process_batch(ch, batch)`  
  [_process_batch](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L39-L66)
- channel 消费（触发 agent 的瞬间）：`BaseChannel._run_process_loop` 中执行 `async for event in self._process(request)`  
  这一行就是“驱动 agent（模型+工具）”的开始  
  [BaseChannel._run_process_loop](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L865-L914)

### 4) 并发模型（报告里最容易写错的点）

- 线程（Thread）是 OS 级并发；协程（coroutine/Task）是同一线程内由 event loop 调度的并发。
- `enqueue` 的 thread-safe 含义：把回调投递到“主 event loop 所在的线程”执行，而不是“切回某个协程”。  
  [ChannelManager.enqueue](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L352-L363)
- 多线程 + 多 loop 的使用场景：例如 weixin long-poll 在后台线程自建 loop，只负责接收；业务处理始终回到主 loop 串行消费。  
  [WeixinChannel.start](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L1426-L1479)

### 5) 新增渠道的“最小实现清单”（可直接放到报告结尾）

- 必做：`channel` 唯一 key、`start/stop`、`build_agent_request_from_native`、`send`
- 建议：覆盖 `resolve_session_id`（群聊/私聊隔离）、覆盖 `send_content_parts`（多媒体原生发送）
- 决策：`uses_manager_queue=True`（默认推荐），否则你要自己保证串行、合并、错误处理
- 注册：内置 registry 或 `custom_channels/` 自动发现；如需 webhook，导出 `register_app_routes(app)`  
  [register_custom_channel_routes](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/registry.py#L136-L189)

### 6) 与 nanobot 的类比（用于对比章节）

- QwenPaw 的 “runner/process” 对应 nanobot 的 “AgentLoop.run（从 bus 消费 inbound、向 bus 发布 outbound）”  
  [AgentLoop.run](file:///d:/编程学习记录/nanobot/nanobot/agent/loop.py#L334-L366)
- nanobot channels 更像“把消息投递到 bus / 从 bus 取消息发送”，不直接依赖 runner 概念；QwenPaw channels 则直接调用注入的 `process(request)` 获取 event 流。

### 7) 常见坑位（报告里值得提醒读者）

- 错把 event loop 当协程：loop 是调度器，协程/Task 才是执行体。
- 在接收线程里直接跑 runner：会阻塞接收、引入跨线程/跨 loop 问题、破坏同 session 串行。
- `session_id` 设计不当：群聊串台、私聊串台、或同一会话并发导致上下文竞态。
- `meta` 丢失：回包需要的字段必须沿链路带回（通常放 `payload["meta"]` 并附到 `request.channel_meta`）。

## 下一课预告

Channels 专题到这里就足够让你能读懂并扩展渠道了。后续如果你要把渠道和 Cron/Heartbeat 串起来读，可以从 `CronExecutor` 如何调用 `channel_manager.send_event/send_text` 入手（它是渠道模块的典型使用方视角）。
