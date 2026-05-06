# Module 08 / Lesson 02：Weixin 一条消息的端到端链路（入参/出参/调用方逐节点串联）

## 本课总览（总）

本课只做一件事：假设“微信端发来一条消息”，我们从 weixin 渠道开始，把消息如何进入 QwenPaw、如何进入队列、如何被 runner 处理、以及如何发回微信，完整串起来。

你会看到 Channels 的设计理念不是“写一堆平台适配代码”，而是把系统拆成两段稳定契约：

1. **入站契约**：平台消息 → native payload（dict）→ AgentRequest
2. **出站契约**：runner 事件流（Event）→ Message → OutgoingContentPart → 平台发送

## 学习目标

- 能背出 weixin 链路的主干调用顺序（10–12 个节点）
- 能在每个节点说清：调用方、入参形状、出参/副作用
- 能理解：为什么要把“接收”和“处理”用队列隔离
- 能理解：`meta` 为什么是渠道回包的关键（context_token 等）

## 先修知识

- 建议先读完 Module 08 / Lesson 01（异步心智模型）  
  [Module_08_Lesson_01_async_mentality_for_channels.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_08_Lesson_01_async_mentality_for_channels.md)

## 本节要回答的关键问题

1. weixin 的“平台消息”在代码里是什么形状？
2. native payload 的最小字段有哪些？谁来消费它？
3. native payload 如何变成 AgentRequest？session_id 在哪一步确定？
4. runner 产出的 message event，最终是怎么回写到平台 API 的？

## 端到端链路（分）

下面按“发生顺序”列节点。每一行都包含：调用方 → 被调方法 → 入参 → 出参/副作用。

### 节点 0：渠道启动（为了让后续能收消息）

- 调用方：`ChannelManager.start_all()`
- 被调：`WeixinChannel.start()`
- 入参：无（使用 self 上的 config/状态）
- 出参/副作用：创建 client、记录主 loop、启动后台线程（poll）  
  - [ChannelManager.start_all](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L462-L494)  
  - [WeixinChannel.start](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L1426-L1479)

#### 节点 0 重点（结合我们刚刚的讨论）

- `start_all()` 在主异步入口运行时，会执行 `self._loop = asyncio.get_running_loop()`，把“当前运行 start_all 的 loop”保存为 Channels 体系的主 loop；后续跨线程投递都以这个 loop 为目标。
- `start_all()` 会初始化 `UnifiedQueueManager(consumer_fn=self._consume_queue, queue_maxsize=...)`：它负责按 `(channel_id, session_id, priority_level)` 管理队列，并在需要时自动启动消费者 Task；具体怎么串行消费/批量合并，由 `consumer_fn`（也就是 `_consume_queue`）定义。
- `start_all()` 会对每个启用并且 `uses_manager_queue=True` 的 channel 注入 enqueue 回调：`ch.set_enqueue(_make_enqueue_cb(ch.channel))`。这一步决定了 weixin 的 `self._enqueue(native)` 实际上会走到 `ChannelManager.enqueue("weixin", native)`。
- 回调注入这一段可以用一句话串起来：`start_all()` 把每个 channel 的 `self._enqueue` 指向一个闭包 `cb`；后续 `_on_message` 执行 `self._enqueue(native)` 时，其实就是调用 `cb(native)`，最终把 `native` 投递到 `ChannelManager` 的主 loop 并入队（节点 4–6）。
- `await g.start()` 会真正启动各个 channel：以 weixin 为例，`WeixinChannel.start()` 会记录主 loop、创建 client，并启动 `weixin-poll` 后台线程；该线程只负责 long-poll 拉消息/解析/投递，业务处理（队列消费、runner、回包）仍在主 loop 侧完成。

### 节点 1：poll 线程进入长轮询循环

- 调用方：`WeixinChannel.start()` 启动的线程
- 被调：`WeixinChannel._run_poll_forever()`
- 入参：无
- 出参/副作用：创建新的 event loop，并在该 loop 中运行 `_poll_loop_async()`  
  [weixin/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L426-L496)

### 节点 2：拉取 updates 并遍历平台消息

- 调用方：poll 线程内的 event loop
- 被调：`WeixinChannel._poll_loop_async()`
- 入参：无（内部维护 cursor）
- 出参/副作用：`await client.getupdates(cursor)` 获取 `msgs: List[dict]`；逐条 `await self._on_message(msg, client)`  
  [weixin/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L453-L495)

### 节点 3：解析一条平台 msg → 生成 native payload → 入队

- 调用方：`_poll_loop_async()` 的 `for msg in msgs`
- 被调：`WeixinChannel._on_message(msg, client)`
- 入参：
  - `msg: Dict[str, Any]`（平台原始消息 dict）
  - `client: ILinkClient`
- 出参/副作用：
  - 解析 `item_list` → `content_parts: List[Any]`（TextContent/ImageContent/FileContent/...）
  - 计算 `session_id = self.resolve_session_id(from_user_id, meta)`
  - 构造 `native: dict` 并调用 `self._enqueue(native)`
  - 若 `self._enqueue` 为空：说明未注入 enqueue（通常是 start_all 没跑或 uses_manager_queue 被关）
  [weixin/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L501-L785)

native payload（这是后续统一消费的“输入契约”）建议至少包含：

- `channel_id: str`
- `sender_id: str`
- `session_id: str`
- `content_parts: List[Any]`
- `meta: Dict[str, Any]`

weixin 构造点见：  
[weixin/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L768-L776)

### 节点 4：poll 线程把消息送回主 loop（线程安全）

- 调用方：`WeixinChannel._on_message()` 的 `self._enqueue(native)`
- 被调：`ChannelManager.enqueue(channel_id, payload)`（由 start_all 注入的回调间接调用）
- 入参：
  - `channel_id: str`（"weixin"）
  - `payload: Any`（native dict）
- 出参/副作用：`call_soon_threadsafe(self._enqueue_one, channel_id, payload)`，把后续工作切回主 loop  
  [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L352-L364)

### 节点 5：主 loop 分类 priority、提取 session_id，入统一队列管理器

- 调用方：主 event loop（被 `call_soon_threadsafe` 调度）
- 被调：`ChannelManager._enqueue_one(channel_id, payload)`
- 入参：
  - `channel_id: str`
  - `payload: Any`（native dict）
- 出参/副作用：
  - `query = ch._extract_query_from_payload(payload)`（用于命令优先级）
  - `priority_level = command_registry.get_priority_level(query)`
  - `session_id = _extract_session_id(ch, payload)`（优先用 payload['session_id']）
  - 创建 task 调用 `_enqueue_with_timeout(...)` 执行真正 enqueue  
  [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L258-L304)

### 节点 6：真正入队（按 channel+session+priority 建队列）

- 调用方：`_enqueue_one()` 创建的 task
- 被调：`ChannelManager._enqueue_with_timeout(...)`
- 入参：`channel_id, session_id, priority_level, payload, query`
- 出参/副作用：`await _queue_manager.enqueue(channel_id, session_id, priority_level, payload)`  
  [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L305-L351)

### 节点 7：消费者循环启动并串行处理该 session 的消息

- 调用方：`UnifiedQueueManager`（内部启动消费者时调用 consumer_fn）
- 被调：`ChannelManager._consume_queue(queue, channel_id, session_id, priority_level)`
- 入参：
  - `queue: asyncio.Queue`
  - `channel_id/session_id/priority_level`
- 出参/副作用：循环 drain 一批 payload，形成 batch，调用 `_process_batch(ch, batch)`  
  [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L365-L461)

### 节点 8：批合并（可选）→ 进入 BaseChannel 统一消费入口

- 调用方：`_consume_queue()`
- 被调：`_process_batch(ch, batch)`
- 入参：
  - `ch: BaseChannel`（此处是 WeixinChannel 实例）
  - `batch: List[Any]`（native payload list）
- 出参/副作用：
  - `merged = ch.merge_native_items(batch)`（weixin 覆盖，保留 user_id/session_id 等）
  - `await ch._consume_one_request(merged)`  
  - 代码：[_process_batch](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L39-L66)、[WeixinChannel.merge_native_items](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L275-L291)

### 节点 9：native payload → AgentRequest（入站契约落地）

- 调用方：`_process_batch`
- 被调：`BaseChannel._consume_one_request(payload)`
- 入参：`payload`（此时通常是合并后的 native dict）
- 出参/副作用（关键阶段）：
  - 无文本 debounce：媒体先到、文本后到时缓冲合并（避免“只发一张图就触发回复”）
  - `request = self._payload_to_request(payload)`
  - `to_handle = self.get_to_handle_from_request(request)`
  - `send_meta = payload['meta']`（优先来自 payload，保证回包关键信息不丢）
  - `await self._run_process_loop(request, to_handle, send_meta)`  
  [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L797-L863)

其中 `_payload_to_request` 的分支非常重要：

- 如果 payload 已经像 AgentRequest（有 `session_id` 且有 `input`），直接返回
- 否则调用 `build_agent_request_from_native(payload)`  
  [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L658-L668)

### 节点 10：weixin 的 build_agent_request_from_native（native → AgentRequest 的具体映射）

- 调用方：`BaseChannel._payload_to_request`
- 被调：`WeixinChannel.build_agent_request_from_native(native_payload)`
- 入参：`native_payload: Any`（native dict）
- 出参：`AgentRequest`
- 关键字段映射：
  - `session_id`：优先用 payload["session_id"]，否则 `resolve_session_id(sender_id, meta)`
  - `content_parts`：原样作为 Message.content
  - `channel_meta`：被设置到 request 上，用于回包时取 `weixin_context_token` 等  
  [weixin/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L251-L273)

### 节点 11：跑 runner（Event 流）并把 message 回包

- 调用方：`BaseChannel._consume_one_request`
- 被调：`BaseChannel._run_process_loop(request, to_handle, send_meta)`
- 入参：
  - `request: AgentRequest`
  - `to_handle: str`
  - `send_meta: Dict[str, Any]`
- 出参/副作用：
  - `async for event in self._process(request)`（process 是 runner.stream_query）
  - 当 `event.object == "message" and event.status == Completed` 时调用 `on_event_message_completed(...)`  
  [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L865-L921)

### 节点 12：Message → OutgoingContentPart → weixin 平台发送

- 调用方：`on_event_message_completed`
- 被调：`send_message_content(to_handle, message, meta)`
- 入参：
  - `to_handle`
  - `message`（runner 完成态 message event）
  - `meta`（send_meta，包含 context_token 等）
- 出参/副作用：
  - renderer 把 message 转为 parts：`_message_to_content_parts(message)`
  - 调 `send_content_parts(to_handle, parts, meta)` 真正发送  
  [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L1052-L1077)

weixin 覆盖了 `send_content_parts`，所以最终落到：

- 调用方：BaseChannel 的 `send_message_content`
- 被调：`WeixinChannel.send_content_parts(to_handle, parts, meta)`
- 入参：
  - `to_handle`
  - `parts: List[OutgoingContentPart]`
  - `meta`（必须能取到 `weixin_from_user_id` / `weixin_context_token`）
- 出参/副作用：分文本/媒体调用平台 API（split_text 分片，媒体走上传发送）  
  [weixin/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L1066-L1164)

## 关键洞察（为什么这么设计）

- **入站统一**：每个渠道都只需要把平台消息适配为 `AgentRequest`，runner 不关心平台细节
- **出站统一**：runner 只产出 event；渠道决定如何把 parts 发回平台（文本/媒体/分片策略）
- **meta 是“回包生命线”**：平台回包需要的 token/webhook/receive_id 都必须能沿链路传到 send_content_parts
- **队列提供两个能力**：同 session 串行 + 批处理合并（避免竞态与碎片回复）

## 动手练习（可执行）

1. 在 [WeixinChannel._on_message](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L501-L785) 找到 `meta` 构造处，写下你认为“回包必须”的字段（至少 2 个）。
2. 跟踪这两个字段是否能出现在 `send_meta` 中：看 [BaseChannel._consume_one_request](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L797-L863) 的 `send_meta` 选择逻辑。
3. 想象一个 bug：如果 `_on_message` 没有把 `weixin_context_token` 放进 meta，会发生什么？你会在哪个节点打日志定位？

## 验收清单

- [ ] 能把节点 3 → 12 复述出来（不要求背行号，但要背顺序与作用）
- [ ] 能说清 native payload 与 AgentRequest 的区别与转换位置
- [ ] 能说清 meta 的用途（至少举 2 个字段例子）
- [ ] 能解释：为什么队列按 session 串行

## 下一课预告

下一课用 Console 对比 weixin，讲清“哪些东西是 Base 抽象的一部分、哪些是渠道差异”，并总结：新增 channel 的必做/选做清单与验证方式。  
[Module_08_Lesson_03_console_contrast_extension_and_tests.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_08_Lesson_03_console_contrast_extension_and_tests.md)
