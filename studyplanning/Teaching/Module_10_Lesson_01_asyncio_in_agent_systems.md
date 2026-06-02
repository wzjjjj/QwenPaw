# Module 10 / Lesson 01：asyncio 工程化：并发、线程与取消（以 QwenPaw 为例）

## 本课总览（你要学的不是语法，而是“工程边界”）

在 Agent 系统里，异步编程解决的是两个工程问题：

- 并发：同一时刻要“等很多事”（模型推理、网络 IO、工具 IO、队列消费、心跳、后台任务）。
- 可控：超时、取消、资源释放、异常传播必须明确，否则系统会出现卡死、泄漏、乱序或吞错。

本课用 QwenPaw 的真实链路把 asyncio 的“工程化用法”串起来：什么时候用 `async/await`，什么时候必须 `to_thread`，什么时候要“线程 + 自建 event loop”，以及如何正确处理取消与清理。

## 学习目标

- 能在代码里识别异步边界：阻塞 IO / 异步 IO / CPU 密集各自应该怎么跑
- 能解释并复盘一个完整的“取消语义”：谁发起取消、谁接收取消、谁必须清理
- 能看懂项目里“线程 + 自建 event loop”的动机与停机姿势
- 能把一个异步链路补齐：超时 + 取消 + 资源释放（并能用最小测试验证）

## 先修知识

### 1. 核心概念

#### (1) `async def` - 定义协程函数

```python
# 同步函数
def sync_function():
    return "hello"

# 异步函数（协程）
async def async_function():
    return "hello async"
```

协程是轻量级的并发编程方式，可以在单线程内实现多个任务的"并发"执行。

#### (2) `await` - 暂停并让出控制权

`await` 用于暂停当前协程，等待另一个异步操作完成，**期间 event loop 可以去执行其他任务**。

```python
async def fetch_data():
    # await 等待期间，event loop 继续处理其他请求
    result = await asyncio.to_thread(sync_api_call, "https://example.com")
    return result
```

**关键理解**：`await` 暂停的是**当前协程**，不是整个 event loop！其他任务仍然可以正常执行。

#### (3) `asyncio.create_task` - 创建后台任务

当需要让一个协程**后台运行**，不阻塞当前流程时使用。

```python
async def _producer():
    async for sse in stream_fn(payload):
        # 广播消息给订阅者
        pass

# 创建后台任务，立即返回，不等待完成
run.task = asyncio.create_task(_producer())
```

| 操作 | 行为 | 适用场景 |
|------|------|----------|
| `await coro()` | 等待协程完成后再继续 | 需要结果才能继续 |
| `asyncio.create_task(coro())` | 立即返回 Task 对象，后台运行 | 无需等待结果，后台执行 |

### 2. `asyncio.to_thread` - 阻塞 IO 的异步化

#### 为什么需要 `to_thread`

如果在协程中直接调用阻塞函数（如同步 HTTP 请求、文件读写），会**卡住整个 event loop**，导致其他任务无法执行。

**错误示例**：
```python
async def bad_example():
    result = sync_api_call()  # ❌ 阻塞！event loop 卡住
    return result
```

**正确做法**：
```python
async def good_example():
    # 将阻塞函数放到线程池执行，不阻塞 event loop
    result = await asyncio.to_thread(sync_api_call)
    return result
```

#### 基本语法

```python
await asyncio.to_thread(func, /, *args, **kwargs)
```

**项目示例**（`agent_management.py:466-470`）：
```python
target_exists = await asyncio.to_thread(
    agent_exists,      # 阻塞函数
    normalized_to_agent,
    None,
)
```

### 3. 工作负载分类与处理策略

| 类型 | 示例 | 处理方式 |
|------|------|----------|
| **异步 IO** | HTTP/WebSocket、异步文件、队列消费 | `await` 直接调用 |
| **阻塞 IO** | 同步 HTTP、子进程等待、第三方 SDK | `asyncio.to_thread()` |
| **CPU 密集** | 压缩/加密/大解析 | 进程池或专用 worker |

### 4. 核心原则：阻塞 IO 会卡住 event loop

#### event loop 的角色

event loop 是 asyncio 的核心调度器，负责管理所有协程的执行顺序。当某个协程执行阻塞操作时，整个调度器会被卡住，其他任务无法执行。

#### 对比：`await` vs 阻塞调用

| 操作 | 是否释放 event loop | 效果 |
|------|---------------------|------|
| `await coro()` | ✅ 是 | 其他任务可以执行 |
| `time.sleep()` | ❌ 否 | 卡住整个 event loop |
| 同步函数调用 | ❌ 否 | 卡住整个 event loop |

### 5. 常见疑问解答

#### 疑问1：为什么要用 `await asyncio.to_thread(agent_exists, ...)` 而不是直接调用？
> **解答**：`agent_exists` 是同步函数，直接调用会阻塞 event loop，影响并发性能。用 `to_thread` 将其放到后台线程执行，释放 event loop 处理其他请求。

#### 疑问2：为什么用 `asyncio.create_task(_producer())` 而不是 `await _producer()`？
> **解答**：`_producer()` 是长时间运行的流式任务（可能永不结束）。如果用 `await`，当前协程会一直等待，后续代码永远执行不到。用 `create_task` 让其在后台运行，当前协程可以立即返回。

#### 疑问3：如果在协程中调用同步数据库查询会怎样？如何修复？
> **解答**：会卡住 event loop。修复方案：使用 `asyncio.to_thread()` 包装，或改用异步数据库驱动（如 `asyncpg`、`asyncmy`）。

#### 疑问4：`await` 期间 event loop 会继续执行吗？
> **解答**：**会！** `await` 的本质就是"暂停当前协程，让出控制权"，让 event loop 调度其他任务。这是异步编程实现并发的关键。

## 本节要回答的关键问题

1. 为什么工具函数里会出现 `await asyncio.to_thread(...)`？这和直接 `await` 有什么本质区别？
2. 为什么同一项目里既有“主 event loop”，又有“后台线程里自建 event loop”（例如 weixin）？
3. `CancelledError` 什么时候应该吞掉，什么时候必须向上抛出？
4. 资源释放应该写在哪里：`try/finally`、`async with`、还是 context manager？

## 核心概念（只讲工程里必须用的那部分）

### 1) 你要先划分三类工作负载

- 异步 IO：HTTP/WebSocket、异步文件、队列消费等，优先 `async/await`
- 阻塞 IO：同步 HTTP、子进程等待、某些第三方 SDK，只能 `to_thread` 或专用线程池
- CPU 密集：压缩/加密/大解析，优先进程池或专门 worker，避免污染 event loop

QwenPaw 的一个典型“阻塞 IO → 异步化”做法是把同步函数放到线程里跑：

- [list_agents](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L412-L423)（`asyncio.to_thread` 包装同步请求）
- [chat_with_agent](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L426-L499)（多个阻塞步骤拆分到 `to_thread`）

### 2) 取消不是“异常处理”，而是“控制协议”

取消（`asyncio.CancelledError`）在工程里意味着：“上层决定这条链路必须停止”。你要保证两件事：

- 该停止的停止：不要吞掉导致任务继续跑（浪费资源/产生副作用）
- 该清理的清理：临时文件、子进程、订阅队列、后台 task 必须释放

QwenPaw 里可以直接观察到“取消 + 清理”的完整形态：

- [TaskTracker.request_stop](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/task_tracker.py#L189-L213)（显式 `task.cancel()`）
- [TaskTracker.unregister_external_task](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/task_tracker.py#L132-L153)（结束时给订阅者发 sentinel + resolve future）

### 3) 为什么会出现“线程 + 自建 event loop”

现实渠道 SDK 或长轮询逻辑经常要求：

- 独立于主 loop 运行（避免阻塞 FastAPI/runner 的主事件循环）
- 同步/第三方回调线程模型不友好（必须隔离）

weixin 的长轮询就是典型做法：后台线程创建 event loop，专门跑一个无限循环；关闭时要 cancel pending tasks 并 shutdown async gens：

- [WeixinChannel._run_poll_forever](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L448-L474)
- [WeixinChannel._poll_loop_async](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L475-L518)

### 4) “吞掉取消”是有条件的：必须能解释“谁取消了谁”

QwenPaw 在 MCP client 注册里有一个很工程化的取消处理：只吞掉“内部会话中断导致的取消”，但不吞掉“外部任务取消”。

- [register_mcp_clients](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L478-L560)

你应该能读懂并解释这类模式：为什么要区分、区分的判断依据是什么、吞掉后系统还能不能保持一致性。

## 代码走读路线（建议按这个顺序）

- 工具层：阻塞 IO 如何在工具函数里异步化  
  - [agent_management.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L412-L499)
- Runner 层：如何跟踪/取消后台任务，避免“幽灵任务”  
  - [task_tracker.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/task_tracker.py#L34-L213)
- Agent 层：取消/异常如何在组装阶段被处理（MCP 恢复逻辑）  
  - [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L478-L560)
- Channel 层：线程与 event loop 的边界（weixin）  
  - [weixin/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py#L448-L518)

## 动手练习

### 练习 1（基础）：给“阻塞点”做标注与分类

- 在 [agent_management.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L412-L499) 里列出所有潜在阻塞点，标注为：
  - 网络阻塞 / 文件阻塞 / CPU 密集 / 不确定（第三方库）
- 写出你对每个点的处理策略：`await` / `to_thread` / 单独 worker

### 练习 2（进阶）：为一条异步链路补齐“超时 + 取消 + 清理”

- 选择一个会长时间等待的协程链路（例如 channel 的 poll loop、或某个工具调用）
- 加入超时（`asyncio.wait_for` 或等价机制）并设计取消后的清理动作（`finally` / `async with` / context manager）
- 要求：取消后不留下 pending task、不会重复执行副作用

### 练习 3（挑战）：写一个最小异步测试验证取消语义

- 以 [TaskTracker.wait_all_done](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/task_tracker.py#L83-L102) 或 [TaskTracker.request_stop](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/task_tracker.py#L189-L213) 为对象
- 写一个 pytest-asyncio 测试：构造一个永不结束的任务，触发 stop/timeout，断言最终状态正确（idle/queue sentinel/任务 done）

## 验收清单

- [ ] 我能在 QwenPaw 里指出至少 3 个“阻塞 IO → to_thread”的真实位置，并解释原因
- [ ] 我能解释 `CancelledError` 的传播原则：什么时候必须抛出、什么时候允许吞掉
- [ ] 我能复述 weixin 的线程+event loop 关闭流程，并说明每一步要解决什么风险
- [ ] 我能写出一个最小异步测试，验证“取消后无泄漏、无悬挂任务”

## 下一课预告

下一课进入“类型注解的工程落地”：你会用 QwenPaw 的真实数据结构（tool 输出、provider 配置、channel payload）把类型系统用成“可读性与可维护性”的工具，而不是形式主义。
