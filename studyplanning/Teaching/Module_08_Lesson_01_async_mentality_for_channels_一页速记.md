# Module 08 / Lesson 01：Channels 异步心智模型一页速记（脑图版）

## 总体目标

- 让 Agent 在不同外部系统里：收消息 → 串行处理 → 回消息
- 抗阻塞、抗突发高并发、同一会话不乱序

## 四层边界（从外到内）

### A. 接收层（平台/SDK/回调/长轮询）

- 职责：拉取/接收消息、解析成 native payload、尽快返回
- 禁止做：直接跑 LLM/工具（不可控耗时会拖死接收）

### B. 跨线程投递层（thread → 主 loop）

- 核心语义：把“回调”安全投递到主 event loop 所在线程执行
- 关键动作：`call_soon_threadsafe(self._enqueue_one, channel_id, payload)`
- 关键代码示例（ChannelManager.enqueue）：

```python
def enqueue(self, channel_id: str, payload: Any) -> None:
    """Enqueue a payload for the channel. Thread-safe (e.g. from sync
    WebSocket or polling thread). Call after start_all().
    """
    if self._loop is None:
        logger.warning("enqueue: loop not set for channel=%s", channel_id)
        return
    self._loop.call_soon_threadsafe(
        self._enqueue_one,
        channel_id,
        payload,
    )
```

#### 学习笔记：怎么理解“回调 / 投递到主 loop / poll 线程”

- 回调（callback）：本质是“函数引用 + 参数”。这里就是把未来要执行的 `_enqueue_one(channel_id, payload)` 交给 event loop 去调用。
- `call_soon_threadsafe` 做的不是“把外部线程加入主 loop”，而是“把一个回调投递到主 loop 的待执行队列里，并唤醒主 loop”。真正执行 `_enqueue_one(...)` 的仍然是主 loop 所在线程。
- `self._loop` 为什么被视为主 loop：它在 `ChannelManager.start_all()` 里通过 `asyncio.get_running_loop()` 捕获并保存；工程上约定 start_all 在主异步入口运行，所以这里拿到的就是 Channels 体系选定的主 loop。
- poll 线程是什么意思：某些平台用 long-poll/polling 拉消息，会在后台线程里“等很久 + 反复请求平台”；它只负责收/解析/投递，业务处理必须回到主 loop。

### C. 队列治理层（入队/路由/串行/批量合并）

- QueueKey：`(channel_id, session_id, priority_level)`
- 同一 QueueKey：单消费者 Task 串行消费
- 批量合并：先 `await queue.get()` 拿第一条，再 `get_nowait()` 吸走已到达的同队列消息组成 batch
- 关键代码示例（路由入队：ChannelManager.\_enqueue\_one）：

```python
def _enqueue_one(self, channel_id: str, payload: Any) -> None:
    if self._queue_manager is None:
        logger.warning(
            "enqueue: queue_manager not initialized for channel=%s",
            channel_id,
        )
        return

    ch = next(
        (c for c in self.channels if c.channel == channel_id),
        None,
    )
    if not ch:
        logger.warning(
            "enqueue: channel not found: channel_id=%s",
            channel_id,
        )
        return

    query = ch._extract_query_from_payload(payload)
    priority_level = self._command_registry.get_priority_level(query)
    session_id = self._extract_session_id(ch, payload)

    task = asyncio.create_task(
        self._enqueue_with_timeout(
            channel_id,
            session_id,
            priority_level,
            payload,
            query,
        ),
    )
    self._enqueue_tasks.add(task)
    task.add_done_callback(self._enqueue_tasks.discard)
```

- 关键代码示例（创建队列 + 启动 consumer：UnifiedQueueManager.enqueue / \_get\_or\_create\_queue）：

```python
async def enqueue(
    self,
    channel_id: str,
    session_id: str,
    priority_level: int,
    payload: Any,
) -> None:
    queue_key = (channel_id, session_id, priority_level)
    state = await self._get_or_create_queue(queue_key)
    state.last_activity = time.time()
    await asyncio.wait_for(state.queue.put(payload), timeout=30.0)

async def _get_or_create_queue(self, queue_key: QueueKey) -> QueueState:
    async with self._lock:
        if queue_key in self._queues:
            return self._queues[queue_key]

        queue = asyncio.Queue(maxsize=self._queue_maxsize)
        channel_id, session_id, priority_level = queue_key
        consumer_task = asyncio.create_task(
            self._run_consumer(
                queue,
                channel_id,
                session_id,
                priority_level,
            ),
            name=(
                f"consumer_{channel_id}_"
                f"{session_id[:20]}_{priority_level}"
            ),
        )
        state = QueueState(queue=queue, consumer_task=consumer_task)
        self._queues[queue_key] = state
        return state
```

- 关键代码示例（串行消费 + batch drain：ChannelManager.\_consume\_queue）：

```python
while True:
    payload = await queue.get()

    ch = await self.get_channel(channel_id)
    if not ch:
        for _retry in range(3):
            await asyncio.sleep(0.5)
            ch = await self.get_channel(channel_id)
            if ch:
                break
        if not ch:
            logger.error(
                "Consumer: channel not found after retries: channel_id=%s",
                channel_id,
            )
            return

    batch = [payload]
    while True:
        try:
            next_payload = queue.get_nowait()
            batch.append(next_payload)
        except asyncio.QueueEmpty:
            break

    await _process_batch(ch, batch)
```

- 关键代码示例（batch 合并：\_process\_batch）：

```python
async def _process_batch(ch: BaseChannel, batch: List[Any]) -> None:
    if len(batch) > 1 and ch._is_native_payload(batch[0]):
        merged = ch.merge_native_items(batch)
        await ch._consume_one_request(merged)
    elif len(batch) > 1:
        merged = ch.merge_requests(batch)
        if merged is not None:
            await ch._consume_one_request(merged)
        else:
            await ch.consume_one(batch[0])
    elif ch._is_native_payload(batch[0]):
        await ch._consume_one_request(batch[0])
    else:
        await ch.consume_one(batch[0])
```

### D. 业务执行层（Channel consume → runner.process → event 流 → send）

- runner.process 是 async generator：`async for event in process(request)` 持续产出事件
- BaseChannel 将事件映射为平台回包并发送
- 关键代码示例（consume 入口：BaseChannel.consume\_one）：

```python
async def consume_one(self, payload: Any) -> None:
    if self._debounce_seconds > 0 and self._is_native_payload(payload):
        key = self.get_debounce_key(payload)
        self._debounce_pending.setdefault(key, []).append(payload)

        old = self._debounce_timers.pop(key, None)
        if old and not old.done():
            old.cancel()

        async def flush(k: str) -> None:
            await asyncio.sleep(self._debounce_seconds)
            items = self._debounce_pending.pop(k, [])
            self._debounce_timers.pop(k, None)
            if not items:
                return
            merged = self.merge_native_items(items)
            if not merged:
                return
            await self._consume_one_request(merged)

        self._debounce_timers[key] = asyncio.create_task(flush(key))
        return
    await self._consume_one_request(payload)
```

- 关键代码示例（组装 request 并进入 process loop：BaseChannel.\_consume\_one\_request）：

```python
request = self._payload_to_request(payload)
if isinstance(payload, dict):
    meta_from_payload = dict(payload.get("meta") or {})
    if payload.get("session_webhook"):
        meta_from_payload["session_webhook"] = payload["session_webhook"]
    setattr(request, "channel_meta", meta_from_payload)
to_handle = self.get_to_handle_from_request(request)
await self._before_consume_process(request)
send_meta = getattr(request, "channel_meta", None) or {}
await self._run_process_loop(request, to_handle, send_meta)
```

- 关键代码示例（驱动事件流：BaseChannel.\_run\_process\_loop）：

```python
async for event in self._process(request):
    obj = getattr(event, "object", None)
    status = getattr(event, "status", None)
    if obj == "content":
        if await self.on_event_content(request, to_handle, event, send_meta):
            continue
    if obj == "message" and status == RunStatus.Completed:
        await self.on_event_message_completed(
            request,
            to_handle,
            event,
            send_meta,
        )
    elif obj == "response":
        last_response = event
        await self.on_event_response(request, event)
```

## 核心术语对齐（读代码唯一正确口径）

- 线程（thread）：OS 调度单元；可真正并行
- event loop：调度器，运行在某个线程里
- coroutine：`async def` 产生的可等待对象；不自动跑
- Task：协程的运行实例（提交给某个 loop 执行）
- call\_soon\_threadsafe：跨线程把回调塞进“指定 loop”的待执行队列并唤醒它

## 为什么要“入队 + 按 session 串行”

- 解决三类典型 bug：
  - 回复乱序（先完成的不一定先发送）
  - 上下文/记忆/状态竞态（并发写同一 session）
  - 工具副作用竞态（重复执行/互相覆盖）
- 顺带解决：媒体先到、文本后到 → batch merge 更容易拼成完整输入

## 为什么 weixin 用“后台线程 + 自建 event loop”

- long-poll 等很久：不能占主 loop
- poll 自己需要 async I/O：独立 loop 更干净
- 严格边界：poll 线程只做“收/解析/投递”；业务处理永远回到主 loop
- 关键代码示例（起线程：WeixinChannel.start）：

```python
self._loop = asyncio.get_running_loop()
self._stop_event.clear()

self._poll_thread = threading.Thread(
    target=self._run_poll_forever,
    daemon=True,
    name="weixin-poll",
)
self._poll_thread.start()
```

- 关键代码示例（线程内 new loop 跑 poll：WeixinChannel.\_run\_poll\_forever）：

```python
def _run_poll_forever(self) -> None:
    if sys.platform == "darwin":
        poll_loop = asyncio.SelectorEventLoop()
    else:
        poll_loop = asyncio.new_event_loop()
    asyncio.set_event_loop(poll_loop)
    self._poll_loop = poll_loop
    poll_loop.run_until_complete(self._poll_loop_async())
```

- 关键代码示例（poll loop 拉消息并交给 \_on\_message：WeixinChannel.\_poll\_loop\_async）：

```python
while not self._stop_event.is_set():
    data = await client.getupdates(cursor)
    msgs: List[Dict[str, Any]] = data.get("msgs") or []
    for msg in msgs:
        await self._on_message(msg, client)
```

- 关键代码示例（解析后入队：WeixinChannel.\_on\_message）：

```python
native = {
    "channel_id": self.channel,
    "sender_id": from_user_id,
    "user_id": "" if is_group else from_user_id,
    "session_id": session_id,
    "content_parts": content_parts,
    "meta": meta,
}
if self._enqueue is not None:
    self._enqueue(native)
```

## 一条消息的端到端链路（背下来就不乱）

- 平台消息到达（可能在 poll 线程）→ 解析 native → `ChannelManager.enqueue`
- `call_soon_threadsafe` 投递 `_enqueue_one` 到主 loop
- 主 loop：按 QueueKey 入队（必要时创建 consumer Task）
- consumer Task：按 session 串行取出 → batch merge → `BaseChannel.consume_one`
- `BaseChannel._run_process_loop`：`async for event in runner.process(request)` → 发送回平台
