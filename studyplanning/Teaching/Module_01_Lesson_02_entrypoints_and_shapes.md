# Module 01 / Lesson 02：入口与数据形状（Entrypoints & Shapes）

## 学习目标

- 能列出 QwenPaw 的 4 类主要入口（CLI / Console API / Runtime API / Channels），并说明它们如何汇聚到同一套 runner/agent 执行链路
- 能把一次请求的关键字段（agent_id、session_id、root_session_id、channel、user_id、input blocks）整理成“形状表”，并指出每个字段来自哪里、在哪里被使用
- 能用“输入 shape → 变换点 → 输出/副作用”讲清一条最短主链路（推荐从 Console `/console/chat` 开始）

## 先修知识

- FastAPI 基础：路由、中间件、Request/Response、StreamingResponse
- Python async：async generator、任务取消（CancelledError）
- 对 SSE（Server-Sent Events）有概念即可（不需要会前端）

## 本节要回答的关键问题

1. “入口”为什么是工程里最重要的学习抓手？入口在帮系统解决什么“不确定性”？
2. QwenPaw 的 `agent_id` 是怎么注入并传到 runner 的？为什么既支持 path 又支持 header？
3. `session_id` 与 `root_session_id` 分别服务什么目的？为什么需要 root_session_id？
4. Console 为什么要引入 TaskTracker？它解决了“流式断线重连”的哪几个问题？

## 核心概念（把“形状”当成契约）

读一个 agent 系统，第一件事不是读模型，而是读 **输入/输出形状（shape）**：

- shape 是“系统边界”最硬的东西：你改内部实现没问题，但一旦 shape 变化，上层/下游就会碎。
- shape 是“调试定位”最短路径：很多 bug 本质是字段缺失、字段来源错误、字段在链路中被覆盖/丢失。

本节只记住一个套路：
- 每过一层，都写一句：这一层“接收什么 shape / 输出什么 shape / 产生什么副作用”。

## 代码走读路线（建议按这个顺序）

### 1) CLI 入口：`python -m qwenpaw` 与 click 的 lazy subcommands

- 入口：[`__main__.py`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/__main__.py)
- CLI 主体：[`cli/main.py`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/cli/main.py#L58-L172)

你需要关注的点：
- `LazyGroup` 如何做到“子命令按需 import”，以及它为什么要记录 init timings（提示：启动性能/Windows 兼容）
- CLI 上的 `--host/--port` 是怎么从 “last run” 回填默认值的（这类机制往往影响排障）

### 2) API 入口：agent-scoped middleware 如何注入 `agent_id` / `root_session_id`

- 中间件：[`AgentContextMiddleware`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L15-L63)

关键 shape 字段：
- `agent_id`：优先从 `/api/agents/{agentId}/...` 解析，其次从 `X-Agent-Id`
- `root_session_id`：从 `X-Root-Session-Id` 读取，并写入 `request.request_context`

这一步解决的是“一个进程服务多个 agent”时的隔离问题：后续 runner/workspace 不再通过参数层层传递，而是通过上下文变量取当前 agent。

### 3) Runtime 通用链路：DynamicMultiAgentRunner 把请求路由到 workspace.runner

- 动态 runner：[`DynamicMultiAgentRunner`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L193)
- workspace 懒加载：[`MultiAgentManager.get_agent`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)

你需要抓住的 shape 变换点：
- 输入：FastAPI request（其中 agent_id 来自 contextvar）
- 输出：`async for item in runner.stream_query(...)` 的流式 chunk（最终作为 SSE 输出）
- 副作用：在 `finally` 中 unregister 外部任务，保证 reload/停机可以判断“是否还有 in-flight task”

### 4) Console 链路：`POST /console/chat` 的 request shape 提取与 TaskTracker

- Console chat 入口：[`post_console_chat`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/console.py#L128-L217)
- shape 提取：[`_extract_session_and_payload`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/console.py#L65-L99)
- 断线重连/多订阅/stop：TaskTracker（后续课会系统讲 queue 语义）：[`task_tracker.py`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/task_tracker.py#L34-L220)

Console 这条链路非常适合作为“shape 练习题”的主样例，因为它把：
- `AgentRequest` / dict 两种输入都兼容掉了
- 原始输入 content 被抽成了 `native_payload`（包含 meta/session_id/user_id）
- 流式输出不直接挂在 request 上，而是转为 TaskTracker 的 run_key（chat.id）驱动

## 关键形状表（你要能背下来的最小字段集）

| 字段 | 典型来源 | 典型落点（使用位置） | 备注 |
| --- | --- | --- | --- |
| agent_id | path `/api/agents/{agentId}` 或 `X-Agent-Id` | `MultiAgentManager.get_agent(agent_id)` | 决定 workspace 隔离边界 |
| session_id | `AgentRequest.session_id` 或 Console dict | Console `resolve_session_id(...)` / runner session | 决定对话/状态落盘的“会话维度” |
| root_session_id | `X-Root-Session-Id` | 请求上下文（用于跨会话审批/路由等） | 你通常把它当“跨线程索引” |
| channel | `AgentRequest.channel` 或默认 `"console"` | channel_manager / runner 构造 request_context | 决定工具过滤、渲染与事件规范化 |
| user_id | `AgentRequest.user_id` 或 Console dict | chat/session 文件名的一部分、权限策略 | Windows 文件名敏感（下一课） |
| input blocks | `AgentRequest.input[*].content[*]` | `_extract_session_and_payload` 抽取为 content_parts | 文本/媒体 block 混合，需要统一抽象 |

## 动手练习

### 练习 1（基础）：画“入口矩阵”

产出一张表格（可以直接写在笔记里），列出：
- CLI / Console API / Runtime API / Channels 各自的入口文件与核心函数
- 它们最终如何汇聚到 workspace.runner（是直连还是经 TaskTracker/ChannelManager）

验收：
- 每个入口至少给出 1 个可点击代码锚点链接（路径 + 行号范围）

### 练习 2（进阶）：补齐一份“shape 字段表”

以 Console `/console/chat` 为样例，从请求体里找出：
- channel_id / sender_id(user_id) / session_id / content_parts

并回答：
- 哪些字段是“用户可控输入”（需要做校验/默认值）？
- 哪些字段是“系统注入”（例如 agent_id/root_session_id）？

验收：
- 你的字段表至少覆盖本节“关键形状表”中的 6 个字段

## 验收清单

- [ ] 能讲清 4 类入口各自的职责边界，并说出“它们如何汇聚到 runner/agent”
- [ ] 能指出 `agent_id` 注入的优先级与代码位置
- [ ] 能解释 Console 为什么需要 TaskTracker（至少说出 buffer/reconnect/stop 三个关键词）
- [ ] 已完成“入口矩阵”与“shape 字段表”两份产出物

## 下一课预告

下一课把“形状”落到工程稳定性上：配置与会话持久化的失败语义（缺文件/坏 JSON/校验失败/Windows 文件名非法/并发写损坏），以及系统是如何降级/恢复的。

入口文件：
- [`load_config`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/utils.py#L538-L576)
- [`SafeJSONSession`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/session.py#L78-L176)

