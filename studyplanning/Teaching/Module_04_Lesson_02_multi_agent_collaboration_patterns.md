# Module 04 / Lesson 02：多智能体协作的工程落地（机制 + 模式 + 反模式）

## 本课总览（总）

很多人说“多智能体协作”，脑子里是“多个模型一起聊天”。但在工程里，协作首先是**运行时与边界**问题：谁拥有状态、谁能写文件、谁能触发副作用、谁负责最终输出，以及它们如何通信、如何追踪会话与任务。

QwenPaw 的“多智能体”有两层含义：

- **运行时多智能体（MultiAgentManager + Workspace）**：同一套服务进程里，同时维护多个 agent 的 workspace（配置/技能/工具开关/会话状态/后台任务），通过 `agent_id` 做路由隔离。
- **协作多智能体（inter-agent chat tools + skills）**：某个 agent 在一次对话中，需要另一个 agent 的专长时，通过内建工具 `list_agents/chat_with_agent/submit_to_agent/check_agent_task` 发起“对另一个 agent 的 API 调用”，把对方当成一个可调用的子系统。

这一课把这两层都讲清楚，并给你一套可复用的协作模式清单：你在银行的 agent 项目里，不需要“追求多智能体”，但你需要**知道什么时候该拆、怎么拆、怎么收口**。

## 学习目标

- 能说清 QwenPaw “多智能体”两层语义：运行时隔离 vs 协作通信
- 能画出 inter-agent chat 的端到端链路（Agent A → /agent/process → Agent B → SSE → A 汇总）
- 能用 QwenPaw 内建工具实现两种协作：前台求助（同步拿结果）与后台委派（异步拿 task）
- 能掌握 5 类可落地协作模式，并识别常见反模式（循环对话、甩锅、状态错位、权限错配）

## 先修知识

- 已理解：`agent_id` / `session_id` 的基本概念
- 允许对“多智能体协作”零基础，本课会从 QwenPaw 的具体实现倒推抽象

## 本节要回答的关键问题

1. QwenPaw 的“多个 agent”到底隔离了什么？共享了什么？
2. 一个 agent 如何“调用另一个 agent”？它到底是工具调用还是 HTTP 调用？
3. `session_id` 与 `root_session_id` 在跨 agent 协作里各自解决什么问题？
4. 什么样的协作模式是“稳定可控的”？什么是反模式？

## 核心概念（分）

### 1) 运行时多智能体：Workspace 才是隔离单元

QwenPaw 的 multi-agent 不是“一个请求里并发跑多个模型”，而是“一个服务进程里管理多个 Workspace”，每个 workspace 有自己的：

- agent 配置（模型、tools/skills 开关、渠道配置等）
- 会话状态（memory/context/session）
- 任务追踪（background tasks / reload 时的存活检测）

MultiAgentManager 通过懒加载 + 并发去重来创建 workspace：第一次有人请求该 `agent_id`，才 `Workspace(...).start()`；并发情况下用 `Event` 让后来的请求等待同一个启动过程。

- 入口实现：[MultiAgentManager.get_agent](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)

### 2) 路由隔离：请求如何落到正确的 agent/workspace

要让“同一个 FastAPI 服务”同时服务多个 agent，关键是把 `agent_id` 绑定到当前请求上下文，然后动态选择 workspace runner。

- agent_id 注入（path/header）：[AgentContextMiddleware](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L15-L63)
- 动态 runner 路由（按当前 agent_id 找 workspace）：[DynamicMultiAgentRunner](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L193)

你可以把它理解为“同一套 API，按 `X-Agent-Id` 进入不同租户的运行时”。

### 3) 协作多智能体：inter-agent chat 本质是“对同一服务的二次调用”

QwenPaw 让 agent 可以像调用工具一样“调用另一个 agent”。工程上它不是神秘机制，而是：

1. Agent A 在推理中调用内建工具 `chat_with_agent(...)`
2. 工具内部构造一个 `session_id`（用于与 Agent B 的会话续聊），再向本地 API 的 `/agent/process` 发起请求，并把 `X-Agent-Id` 设为 Agent B
3. 服务端 middleware 将该请求绑定到 Agent B 的上下文，然后 Dynamic runner 路由到 B 的 workspace
4. B 的 runner 创建 per-request 的 QwenPawAgent 实例执行，产出 SSE 流
5. A 侧工具收集 SSE 的最后一条消息，提取文本内容，返回给 Agent A 继续推理与整合

对应关键实现点：

- 工具实现（前台等待结果）：[chat_with_agent](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L426-L507)
- 构造请求体（session_id + 文本前缀）：[build_agent_chat_request](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L189-L230)
- 通过 header 把目标 agent 传给服务端路由：[ _request_headers ](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L232-L246)
- 最终收集 SSE 输出：[collect_final_agent_chat_response](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L275-L297)

### 4) 前台求助 vs 后台委派：同步结果与异步任务

多智能体协作经常会遇到“另一个 agent 要跑很久”，如果你在前台硬等，就会卡住你的主链路。因此 QwenPaw 把协作拆成两种工具路径：

- **前台对话**（马上拿回复）：`chat_with_agent(...)`
- **后台委派**（立刻返回 task_id）：`submit_to_agent(...)`，之后用 `check_agent_task(...)` 轮询结果

对应实现：

- 后台提交：[submit_to_agent](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L509-L584)
- 查询状态：[check_agent_task](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L587-L622)

### 5) root_session_id：跨 agent 协作时的“审批串联”

当系统存在“工具审批/权限守卫”时，协作会引入一个非常现实的问题：**子 agent 触发审批时，审批应该回到谁的会话里？**

QwenPaw 用 `root_session_id` 做“跨会话的归属锚点”：

- middleware 从 `X-Root-Session-Id` 注入 request_context：[agent_scoped.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L49-L60)
- contextvars 里维护 root_session_id：[agent_context.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/agent_context.py#L21-L31)
- inter-agent 工具把当前 root_session 透传给对方请求：[chat_with_agent](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L476-L492)

你可以把它理解为：主控会话是“审批与审计的归属”，子 agent 只是执行者。

### 6) 可落地协作模式清单（建议你优先用这些）

把协作模式当成“组织结构”，不是 prompt 技巧。下面 5 类最常用，也最容易工程化：

1. **Consult（咨询/第二意见）**：主控 agent 问专用 agent 一个明确问题，拿回结论再继续。
2. **Delegate（委派/后台）**：主控 agent 把长任务交给执行型 agent 后台跑，自己继续处理主线，稍后收结果。
3. **Supervisor + Specialists（监督者 + 专家）**：主控 agent 负责拆任务、定义输入输出，专家 agent 负责产出局部结果，主控负责合并与验收。
4. **Reviewer（复核/审计）**：产出型 agent 给初稿，复核型 agent 只做检查与风险提示（不改原稿或只给差异）。
5. **External Delegate（外部 runner 委派）**：当你需要“外部编程/外部工具链”时，用 ACP 把任务交给外部 runner（可选项）。入口工具：[delegate_external_agent](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/delegate_external_agent.py#L444-L589)

QwenPaw 也把这些写进了内建技能说明（建议你把它当作“协作规约模板”）：

- [multi_agent_collaboration-zh/SKILL.md](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/skills/multi_agent_collaboration-zh/SKILL.md)
- [chat_with_agent-zh/SKILL.md](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/skills/chat_with_agent-zh/SKILL.md)

### 7) 反模式与止损机制（你在真实项目里最容易踩的坑）

- **循环对话**：A 问 B，B 再问回 A（或再次调用 B 自己），会放大 token 与成本。QwenPaw 的协作技能里明确写了“不要回调消息来源 agent”（见上面的 SKILL.md）。
- **甩锅式拆分**：主控 agent 不给输入输出契约，只丢一句“你去做”，最后无法验收也无法合并。
- **状态错位**：子 agent 的输出没有绑定到主控会话（session_id/root_session_id 没传好），导致审批/记录散落。
- **权限错配**：执行型 agent 拿到了不该拥有的工具权限；反过来，审计/复核 agent 却能触发副作用。

## 代码走读路线（分）

按“路由隔离 → 协作通信 → 会话串联”的顺序读：

1. 运行时隔离单元与懒加载并发去重：  
   - [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)
2. agent_id 如何进入请求上下文：  
   - [agent_scoped.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L15-L63)
3. 同一套 AgentApp 如何按 agent_id 动态选择 runner：  
   - [ _app.py ](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L193)
4. inter-agent chat 的请求体与 header：  
   - [agent_management.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L189-L246)
5. inter-agent chat 前台/后台两条路径：  
   - [agent_management.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tools/agent_management.py#L412-L622)

## 动手练习（可执行）

### 练习 1：做一次“咨询型协作”（前台）

目标：让你的主控 agent 向另一个 agent 询问一个明确问题，并把答案整合成最终输出。

1. 列出可用 agents：调用 `list_agents()`
2. 选一个目标 agent（例如 QA agent 或你自己创建的“工具型 agent”）
3. 发起求助：

```text
chat_with_agent(
  to_agent="<target_agent_id>",
  text="请你只回答：在 QwenPaw 的 multi-agent 架构里，agent_id 路由发生在哪两处？给出文件路径与函数名。",
)
```

验收点：你能在返回文本里找到可落地的代码入口，然后自己打开对应文件验证。

### 练习 2：做一次“委派型协作”（后台）

目标：把一个需要较长时间的任务交给另一个 agent 后台跑，并在主控侧轮询拿回结果。

```text
submit_to_agent(
  to_agent="<target_agent_id>",
  text="请在后台整理：inter-agent chat 的完整调用链路（从 chat_with_agent 到 /agent/process，再到 DynamicMultiAgentRunner 路由）。输出成 6 个步骤。",
)

check_agent_task(task_id="<task_id>")
```

### 练习 3：画一张“协作链路时序图”

至少画 7 个节点（A→工具→HTTP→middleware→runner→B→SSE→A），并在图上标注两条关键 header：`X-Agent-Id` 与 `X-Root-Session-Id`。

## 验收清单

- [ ] 能解释“运行时多智能体”与“协作多智能体”的区别，并能在代码里指到入口文件
- [ ] 能说清 inter-agent chat 为什么本质是二次 HTTP 调用，并能指出 `X-Agent-Id` 在哪里被读取
- [ ] 能实现一次前台协作与一次后台协作（知道何时用 chat，何时用 submit）
- [ ] 能列出至少 3 条反模式，并给出止损策略（避免循环、定义契约、收口验收）

## 下一课预告

多智能体协作一旦涉及工具与副作用，马上会遇到治理问题：权限、审批、审计、风险识别。下一课进入安全与治理：  
[Module_05_Lesson_01_security_and_governance.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_05_Lesson_01_security_and_governance.md)

