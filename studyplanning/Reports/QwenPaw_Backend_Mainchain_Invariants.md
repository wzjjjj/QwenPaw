# QwenPaw 后端主链路架构报告（Backbone + Invariants）

本报告只回答一件事：**一句用户输入是如何穿过“入口 → 路由 → workspace → runner → per-request agent → tools/skills → 安全治理”，并在“流式输出/取消/错误”语义下稳定落地的。**

## 1. 系统边界（职责分层，不以框架命名）

- **入口/协议层（HTTP/CLI）**：把外部输入转换成稳定的 request shape，并绑定上下文（agent_id/session_id 等），不做推理与业务编排。
- **路由/隔离层（agent-scoped + workspace）**：把请求路由到某个 agent 的 workspace，管理生命周期与隔离边界（懒加载、reload、stop）。
- **Runner（单次请求生命周期编排）**：处理命令短路、skills 注入、流式输出、取消、错误兜底与最小持久化。
- **Per-request Agent（决策/推理执行体）**：每次请求创建一个干净的 QwenPawAgent 实例，完成 ReAct loop、工具选择与调用。
- **治理层（tool guard / approvals / scanning）**：在工具调用前后介入，把高风险动作变成“可控决策 + 可审计事件”。

## 2. 最短主链路（两条链路，一个 runner）

### 2.1 Runtime 通用链路：`/process` → DynamicMultiAgentRunner → workspace.runner

```text
FastAPI app
  └─ AgentContextMiddleware 注入 agent_id/root_session_id
      └─ AgentApp(/process)
          └─ DynamicMultiAgentRunner.stream_query(request)
              ├─ MultiAgentManager.get_agent(agent_id)  # 懒加载 workspace
              ├─ workspace.task_tracker.register_external_task(run_key)
              └─ async for chunk in workspace.runner.stream_query(...)
                     └─ yield chunk (SSE)
```

- 动态 runner（按 `agent_id` 找 workspace）：[app/_app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L193)
- agent_id 的注入： [agent_scoped.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L15-L63)
- workspace 的懒加载与并发去重： [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)

### 2.2 Console（Web）链路：Console API → TaskTracker（可重连）→ workspace.runner

Console 的特点是：除了把请求交给 runner，还要解决“断线重连/多订阅/后台继续跑/停止任务”等产品问题，因此多了一层 **TaskTracker**。

TaskTracker 的“协议语义”（你需要记住的最少事实）：
- run_key 表示“一次运行”，一个 run_key 可以被多个订阅者 attach（重连/多窗口）。
- 每个 run 有一个 buffer：新订阅者先重放 buffer，再实时接收新事件。
- run 结束后，所有 subscriber queue 收到 sentinel 退出。

核心实现见：[task_tracker.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/task_tracker.py#L34-L220)

## 3. 关键不变量（Invariants）= 现象 + 证据 + 破坏后果

### Invariant 1：每个请求都必须绑定一个明确的 `agent_id`

- 现象：同一个后端进程里可以服务多个 agent，每个 agent 有独立 workspace（配置/技能/工具开关/数据目录）。
- 证据：
  - `AgentContextMiddleware` 优先从路径 `/api/agents/{agentId}/...` 注入，其次从 `X-Agent-Id` header 注入，并写入 contextvar： [agent_scoped.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L15-L63)
  - Dynamic runner 每次请求都会通过 contextvar 取当前 agent_id 再去拿 workspace： [app/_app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L87-L125)
- 破坏后果：
  - agent_id 丢失会导致 workspace 路由失败（返回“Agent not found / not initialized”一类错误），或者更隐蔽地“落到 default agent”，造成数据/配置串用。

### Invariant 2：workspace 懒加载必须是“并发去重”的

- 现象：多个请求同时打到同一个 agent 的首次启动时，不应该触发多次 workspace 初始化（否则会出现多份 service、重复 watcher、重复任务、甚至目录锁冲突）。
- 证据：
  - `MultiAgentManager.get_agent` 的 `_pending_starts` + `asyncio.Event` 机制：第一个请求 claim 启动，其余请求 wait： [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L106)
  - 启动慢路径在 lock 外执行，允许“不同 agent”并行初始化： [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L107-L137)
- 破坏后果：
  - 并发初始化会导致服务重复启动、watcher 重复注册、任务状态错乱，最典型表现是“偶现的重复消息/重复回包/无法 stop”。

### Invariant 3：per-request agent 必须“每次请求创建一次”，workspace 服务可复用

- 现象：Agent 是执行体（含 sys_prompt、toolkit、skills、guard、模型实例），请求之间必须干净；但 memory/context/mcp/chat/channel 等服务更适合在 workspace 维度复用。
- 证据：
  - workspace 启动做的是“加载 agent 配置 + 启动 service_manager”，不是创建每次请求的 QwenPawAgent： [workspace.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/workspace/workspace.py#L351-L388)
  - QwenPawAgent 的初始化在 agent 侧完成：构建 toolkit、注册 skills、构建 sys_prompt、创建 model： [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L100-L185)
- 破坏后果：
  - 如果错误地复用 agent 实例：上下文/工具状态会串（尤其是流式输出队列、工具调用历史、内存压缩标记），导致不可解释的“幻觉式”行为与安全策略绕过风险。

### Invariant 4：流式请求必须可取消，且取消要“能及时停止 agent task”

- 现象：用户点击 stop/断线关闭时，后台不应继续烧 token 或持有资源。
- 证据：
  - runner 内部的可取消流式封装： `_stream_printing_messages_interruptible` 会在外层 cancelled 时主动 cancel agent task： [runner.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L67-L105)
  - TaskTracker 支持 `request_stop`：直接 cancel 对应 run 的 task： [task_tracker.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/task_tracker.py#L189-L213)
  - DynamicMultiAgentRunner 在 finally 中 unregister 外部任务，避免 reload 时判断失真： [app/_app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L127-L193)
- 破坏后果：
  - “点了 stop 但还在跑”会直接造成成本/并发配额被吃光；更严重的是 reload/升级时无法安全停机（存在 in-flight task）。

### Invariant 5：会话持久化必须跨平台且具备“损坏恢复”语义

- 现象：session_id/user_id 会被用作文件名；Windows 对文件名字符集限制更严格；并发写或异常退出可能造成 JSON 尾部垃圾/损坏。
- 证据：
  - 文件名 sanitize： [runner/session.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/session.py#L63-L110)
  - JSON 损坏恢复：先 json.loads，失败则 raw_decode 截取首个合法对象，最坏返回 `{}`： [runner/session.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/session.py#L24-L60)
- 破坏后果：
  - 会话文件读写异常会导致“丢状态/死循环重试/启动即崩”；更隐蔽的是 silent fallback 到空 dict 导致“看起来正常，但历史丢了”。

## 4. 扩展点地图（你应该在哪里“加能力”，在哪里“加治理”）

### 4.1 入口/路由扩展（多 agent / 多租户）

- agent-scoped router：把所有资源挂在 `/agents/{agentId}` 下： [agent_scoped.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L66-L104)
- workspace 生命周期：`Workspace.start/stop`： [workspace.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/workspace/workspace.py#L351-L408)
- 懒加载与热重载：`MultiAgentManager`： [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)

### 4.2 工具/技能扩展（能力面）

- Built-in tool 注册点：QwenPawAgent 构建 toolkit： [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L217-L260)
- Skills：由 skills_manager 在工作区解析并注册到 toolkit（后续课程会把技能加载/覆写/隔离讲清楚）。

### 4.3 渠道扩展（异步链路面）

- ChannelManager 拥有队列与消费循环（channel 自己只实现如何 consume）： [channels/manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L68-L110)

## 5. 推荐“读代码路线图”（最短路径 + 不迷路原则）

读代码时只守两条规则：
- 每跳只问两件事：**输入 shape 是什么？输出/副作用是什么？**
- 每层都写一句“职责边界”：**这层应该做什么/不应该做什么**。

建议走读顺序：
1. 入口与上下文注入： [agent_scoped.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L15-L63)
2. 动态 runner 与外部任务注册： [app/_app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L193)
3. workspace 懒加载与去重： [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)
4. workspace 启动与服务管理： [workspace.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/workspace/workspace.py#L351-L388)
5. runner 的可取消流式封装： [runner.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L67-L105)
6. agent 初始化（toolkit/skills/sys_prompt/model）： [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L100-L185)

