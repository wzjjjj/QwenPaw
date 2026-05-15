# Module 01 / Lesson 01：总体概述（总分总）

## 本课总览（总）

Agent 技术正在从“能聊天的模型”走向“可执行的系统工程”：它不仅要会推理，还要能 **调用工具、管理状态、处理失败、做安全治理**，并在真实环境里长期运行。\
这一课先把全局坐标立起来：你将知道 *Agent 领域现在到哪了*、*这套学习路线怎么学*、以及 *在 QwenPaw 里这些概念具体落在哪些代码入口*。

## 学习目标

- 能用工程语言解释 Agent 技术的关键趋势：从 prompt 到运行时，从单体到多智能体，从“会用工具”到“可治理的工具系统”
- 能说清本课程的结构（总分总）与预期成果，并给自己设定可验证的验收标准
- 能画出 QwenPaw 的主链路，并定位关键入口文件（FastAPI → runner → agent → tools/skills → memory/context → security）

## 先修知识

- Python 基础 + Async 基础（能读懂 async/await）
- FastAPI 基础（路由、中间件、Request/Response）
- 对 LLM 有基本理解（prompt、上下文窗口、工具调用）

## 本节要回答的关键问题

1. “Agent”与“Chatbot/LLM 应用”本质差异是什么？工程上差在哪？
2. 一个可落地的 Agent 系统必须具备哪些模块边界（不以框架命名，而以职责命名）？
3. QwenPaw 把“多智能体”和“单次请求的 agent 实例化”放在哪一层做？为什么？
4. 从代码走读角度，应该从哪里开始读，才能最快建立系统认知？

## 对话记录要点（你问过的关键困惑）

- Web Console 输入一句话后，确实是通过 HTTP 请求后端（当前前端实现走 `POST /api/console/chat`，并携带 `X-Agent-Id`），后端再通过 Runner 走一次“单次请求生命周期”
- `contextvar` 在这里主要用来在异步调用链中传播 `agent_id/session_id/root_session_id`，让日志、token 统计、工具执行、审批等模块能拿到“当前正在服务的是哪个 agent”
- “谁来执行 `stream_query` / runner 的工作”这一点容易混：框架层 `stream_query` 是统一的流式包装器，业务侧主要实现 `query_handler` 来决定要流式产出什么内容
- `MultiAgentManager` 的作用不是做推理，而是做 workspace 生命周期管理：按 `agent_id` 懒加载、并发去重、必要时 reload
- `Workspace.start()` 用在 workspace 第一次被拿到（懒加载）或 reload 时；`start_all()` 启动的是配套服务（runner/memory/context/mcp/chat/channel…），不是创建每次请求的 `QwenPawAgent`
- `QwenPawAgent` 是“每次请求创建一次”的实例，创建位置在 runner 的处理函数内部

### 补充：两条常见入口链路（两条都能跑同一套 Runner）

#### A) Web Console 链路：`POST /api/console/chat`（前端 Chat 页面默认）

```text
浏览器/Console Chat 页面
        │  POST /api/console/chat + X-Agent-Id: <agent_id>
        ▼
post_console_chat() → get_agent_for_request() → Workspace(<agent_id>)
        ▼
ConsoleChannel.stream_one(payload)
        │  self._process(request) == workspace.runner.stream_query(...)
        ▼
TaskTracker.attach_or_start(...) → StreamingResponse(SSE)
```

特点：
- 外层多了一套 Console 的 chat/task 管理：ChatManager、TaskTracker、reconnect/stop、标题生成等。
- SSE 是从 TaskTracker 的队列里读出来的（断线可重连、后台可继续跑）。

#### B) Runtime 通用链路：`POST /api/agent/process`（通用协议入口）

```text
客户端 POST /api/agent/process + X-Agent-Id: <agent_id>
        ▼
AgentContextMiddleware → set_current_agent_id(<agent_id>)
        ▼
AgentApp(/process) → DynamicMultiAgentRunner.stream_query(...)
        ▼
MultiAgentManager.get_agent(<agent_id>) → Workspace(<agent_id>)
        ▼
workspace.runner.stream_query(...) → StreamingResponse(SSE)
```

特点：
- 更“直通”runner：少了 Console 的 chat/task 包装，偏 runtime 协议层。
- SSE 由 AgentApp 直接把 runner 产出的 chunk 格式化输出（没有 TaskTracker 队列转发这一层）。

## 关键问题解答（结合 QwenPaw 的代码落点）

### 1) Agent vs Chatbot/LLM 应用：工程差异在哪里？

- Chatbot 更像“把输入喂给模型再把输出吐出来”，状态与工具常常是“可选项”
- Agent 更像“可执行系统”：除了对话，还需要明确的运行时边界与治理能力
  - 执行入口与生命周期：一次请求如何启动、如何流式输出、如何取消、如何兜底错误（Runner）
  - 工具与扩展：工具/skills/MCP 的注册与热插拔（Agent/ToolKit/MCP Manager）
  - 状态与长期性：会话 state、memory/context 的加载与保存（Session/Memory/Context）
  - 治理与安全：审批、审计、工具守卫引擎（Approval/Tool Guard）

### 2) 一个可落地的 Agent 系统必须具备哪些模块边界？

- **入口层（HTTP/API）**：只负责协议与路由，不负责推理
- **路由层（多智能体/多 workspace）**：把请求绑定到 `agent_id`，并路由到对应 workspace
- **运行时层（Runner）**：编排“单次请求生命周期”，负责流式输出、命令/skill 短路、错误兜底、取消、状态持久化
- **执行体（Per-request Agent instance）**：每次请求创建的 agent，负责主循环（ReAct/plan/工具调用）
- **共享服务（Workspace services）**：runner/memory/context/mcp/chat/channel 等可复用能力，由 workspace 启动并注入给每次请求的 agent
- **治理层（Security/Approvals）**：在工具执行前后介入，做权限、审批、审计、风险控制

### 3) QwenPaw 把“多智能体”和“单次请求的 agent 实例化”放在哪一层做？为什么？

- **多智能体（按 agent\_id 路由 + workspace 生命周期）**：在 `MultiAgentManager + Workspace` 这一层
  - `MultiAgentManager.get_agent()` 懒加载并缓存 workspace（并发去重），首次会 `await Workspace.start()`：\
    [multi\_agent\_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L22-L137)
  - workspace 的启动会 `start_all()` 把配套服务拉起来：\
    [workspace.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/workspace/workspace.py#L348-L384)
- **单次请求的 agent 实例化（QwenPawAgent）**：在 `AgentRunner.query_handler` 内部，每次请求创建一次
  - 这能保证每次请求的执行体干净可控，而 memory/context/mcp 等共享服务复用，性能与隔离更平衡：\
    [runner.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L286-L770)

### 4) 从代码走读角度，应该从哪里开始读，才能最快建立系统认知？

建议用“两条线”交叉读，最不容易迷路：

- **请求主链路（你今天问的这条）**
  1. agent\_id 如何注入并绑定到当前请求上下文：[agent\_scoped.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L15-L63)
  2. 按 agent\_id 路由到 workspace： [\_app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L216)&#x20;
  3. workspace 懒加载与启动（`Workspace.start()`）：[multi\_agent\_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L22-L137)
  4. 单次请求生命周期（`AgentRunner.query_handler`）：[runner.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L286-L770)
- **扩展与治理线（决定“系统可用不可用”的那条）**
  1. MCP 客户端生命周期与热更新：[manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/mcp/manager.py#L23-L165)
  2. Skills 管理与安全扫描入口：[skills\_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/skills_manager.py#L27-L60)
  3. 工具守卫引擎（工具调用前的治理总入口）：[engine.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/security/tool_guard/engine.py#L54-L235)

## 核心概念（分）

### 1) Agent 技术现状与应用前景：从能力到工程化

- **现状**：主流 Agent 项目已经把重点从“prompt 技巧”迁移到“运行时治理”。关键特征包括：
  - 可插拔能力：工具、Skills、外部协议（如 MCP）可热插拔
  - 可控执行：多轮循环（ReAct / plan-execute），能自我纠错但不过度失控
  - 可持久运行：记忆/会话/任务队列/定时任务/多渠道接入
  - 可治理：权限、审批、审计、风险识别、敏感路径防护
- **前景**：Agent 正在成为“智能工作流引擎”。未来竞争点更偏系统能力：
  - 多智能体协作的组织结构（团队/路由/仲裁/监督）
  - 上下文与记忆的工程化（成本、时效、准确性、隐私）
  - 安全与治理（特别是工具调用安全、数据边界与合规）

### 2) 本学习计划的整体框架与预期学习成果

本课程按“总分总”组织：

- **总**：先建立全局坐标（系统边界 + 主链路 + 关键入口）
- **分**：逐个维度拆解（定位/架构/agent/上下文/工具/skills/mcp/记忆/状态/安全/协作/API）
- **总**：用端到端案例把这些维度重新编排成一个可交付系统，并输出可迁移的优化方法

预期学习成果（建议你把它当作验收标准）：

- 能给自己的 Agent 项目写一份简化版设计文档：模块边界 + API 契约 + 状态机 + 安全策略
- 能在 QwenPaw 中定位并解释每个维度对应的实现入口（至少能指到文件与关键函数）
- 能独立完成一个“新增能力”的小改动：新增工具/新增 Skill/新增 MCP client/新增 agent，并知道风险点在哪里

### 3) 学习路径规划与关键技术节点（按工程落点组织）

建议的学习路径（从“系统入口”到“执行内核”再到“扩展与治理”）：

1. **系统入口与多智能体路由**：请求如何绑定 agent\_id，如何路由到不同 workspace
   - 动态 runner 入口：[\_app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L216)
   - agent\_id 注入中间件：[agent\_scoped.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L15-L63)
2. **Runner（一次请求的生命周期）**：命令/skills 短路、流式输出、错误兜底
   - skills 识别与短路：[\_maybe\_inject\_skill](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L185-L259)
   - 单次请求的主处理入口：[`AgentRunner.query_handler`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L286-L770)
3. **Agent 组装与主循环**：工具/skills 注册、sys\_prompt 构建、hooks、MCP 工具注入
   - [react\_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L79-L215)
   - [react\_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L216-L541)
4. **上下文/记忆/状态**：压缩、裁剪、检索、沉淀与可观测状态
   - Context 接口：[base\_context\_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/base_context_manager.py#L15-L223)
   - Memory 接口：[base\_memory\_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/base_memory_manager.py#L18-L116)
5. **扩展与治理（工具/Skills/MCP/安全）**：让系统“可扩展”且“可控”
   - MCP 生命周期管理：[manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/mcp/manager.py#L23-L165)
   - Skills 管理（含安全扫描）：[skills\_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/skills_manager.py#L27-L60)
   - 工具守卫引擎：[engine.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/security/tool_guard/engine.py#L54-L235)

## 代码走读路线（分）

建议你按下面路线走一遍（只看“入口与接口”，不深挖实现）：

1. 看 DynamicMultiAgentRunner 如何按 agent\_id 找到 workspace：[\_app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L126)
2. 看 MultiAgentManager 如何懒加载 workspace（并发去重）：[multi\_agent\_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L22-L137)
3. 看 Runner 如何识别 `/skill` 并短路：[\_maybe\_inject\_skill](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L185-L259)
4. 看 QwenPawAgent 如何注册工具/skills：[\_create\_toolkit/\_register\_skills](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L216-L370)

## 动手练习（可执行）

目标：只做“读代码 + 画图”，不改代码。

1. 画一张“单次请求”的时序图（5 个框就够）：HTTP → agent\_id 注入 → workspace → runner → agent。
2. 画一张“能力扩展”的依赖图：tools / skills / mcp / memory / context / security 分别挂在哪一层。
3. 给每个框写一句“责任边界”，要求可被验收（例如：Runner 不负责长期存储；安全守卫必须在 tool 执行前介入）。

### 进阶练习（提升）

1. 找到“入口层只做 HTTP 语义”的证据：列出 2 个 router 文件，说明它们如何把执行下沉到 workspace/runner（不需要读完实现，能指到调用点即可）。
2. 用你自己的话写出一条“失败语义规则”：什么错误应该在 API 层表现为 4xx，什么错误应该是 5xx，什么错误要回传结构化 detail 让模型自我修复。

## 验收清单

- [ ] 能用 1 分钟讲清：Agent 与普通 LLM 应用在工程上的差异（至少 3 点）
- [ ] 能指出 QwenPaw 的 4 个入口文件，并说明各自职责
- [ ] 有一张自己的主链路图（时序或模块图均可），并能解释层间交互

## 本课小结（总）

你已经把“Agent 工程”从抽象概念落到了可读的代码入口：\
**系统入口/路由** 决定“谁在服务”；**Runner** 决定“一次请求如何跑完”；**Agent 组装** 决定“模型如何被约束为可执行系统”；而 **上下文/记忆/安全** 决定“长期运行是否可控”。\
下一课开始，我们把“要做什么（定位）”和“怎么落地（架构）”对齐，避免后续维度学习变成碎片知识。

## 下一课预告

- 进入 [Module\_02\_Lesson\_01\_positioning\_and\_architecture.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_02_Lesson_01_positioning_and_architecture.md)
