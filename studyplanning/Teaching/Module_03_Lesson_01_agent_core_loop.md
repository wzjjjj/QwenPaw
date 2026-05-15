# Module 03 / Lesson 01：Agent 主体（决策系统 / 推理机制 / 行为模式）（总分总）

## 本课总览（总）

Agent 主体的核心不是“会想”，而是“能把想法变成可控的行动”。  
工程上，你需要把模型包进一个循环（loop），并明确每轮循环的输入、输出、边界与失败模式：**Reasoning → Acting(tool) → Observation → Next Reasoning**。  
这一课以 QwenPaw 的 `QwenPawAgent` 为例，拆解 Agent 主体如何被组装成可运行的系统：工具、skills、sys_prompt、hooks、以及工具守卫（ToolGuardMixin）的 MRO 拦截方式。

## 学习目标

- 能解释 ReAct/Plan-Execute 等行为模式在工程上的共同点与差异点
- 能读懂 QwenPawAgent 的初始化流程，并说清每个组件的职责边界
- 能为自己的 Agent 定义“决策系统”最小闭环：目标 → 计划（可选）→ 行动 → 观察 → 纠错

## 先修知识

- 理解 Module 02 的分层：Agent 层负责“决策与执行循环”，能力层负责“工具/skills/MCP”
- 能读懂 Python 类继承与 MRO

## 本节要回答的关键问题

1. 工程化的 Agent 决策系统包含哪些“必备状态”？哪些应该外置（runner/workspace）？
2. ReAct 为什么适合做“工具调用式执行”？它在工程上最容易失控的点在哪里？
3. QwenPawAgent 为什么把 ToolGuard 做成 Mixin，而不是在工具函数里做检查？
4. Skills 与 Tools 的边界是什么？为什么两者需要共存？

## 核心概念（分）

### 1) 决策系统的工程定义：输入、状态、约束、输出

把 Agent 决策系统写成接口契约，你会更容易做测试与治理：

- **输入**：用户消息 + 运行时上下文（agent_id、channel、会话、环境信息、检索到的记忆）
- **状态**：对话上下文窗口、计划状态（可选）、工具审批状态、任务状态（异步工具）
- **约束**：工具白名单/黑名单、文件访问守卫、预算（token/time/calls）
- **输出**：最终回复 或 工具调用（结构化 tool_use）

在 QwenPaw 中，`QwenPawAgent` 的构造函数就是一次“决策系统组装”：

- 组装工具包（工具函数注册）：[_create_toolkit](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L216-L333)
- 注册 skills（按 workspace + channel 解析有效技能）：[_register_skills](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L335-L370)
- 构建 sys_prompt（从工作目录文件 + 记忆提示 + 环境上下文拼装）：[_build_sys_prompt](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L371-L409)
- 注册 hooks（把上下文压缩/裁剪等横切逻辑接入循环）：[_register_hooks](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L411-L452)

### 2) 推理机制与行为模式：ReAct / Plan-and-Execute 的工程差异

从工程落点看，行为模式主要差在“循环内的状态机”：

- **ReAct**：每轮“思考→工具→观察→再思考”，适合高不确定、需要多次信息采集/操作的任务  
  - 风险：工具调用发散、上下文爆炸、输出不可控  
  - 需要：上下文管理（压缩/裁剪）、工具治理（守卫/审批）
- **Plan-and-Execute**：先计划再执行（计划可变），适合有明确步骤、需要稳定交付的任务  
  - 风险：计划与现实偏离、计划过长、执行死循环  
  - 需要：计划状态机、计划变更策略、失败回滚策略

你会在 QwenPaw 的注释与形态中看到它把“治理”放在 loop 的关键点，而不是靠 prompt：

- 通过 MRO 用 Mixin 截获工具执行：`QwenPawAgent(ToolGuardMixin, ReActAgent)`  
  - [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L79-L97)

### 3) 行为模式落地的关键：把“横切关注点”挂到正确的位置

Agent loop 里最容易出问题的横切关注点包括：

- 上下文过长 → 压缩/裁剪
- 工具输出过大 → 截断或结构化摘要
- 工具调用风险 → 事前守卫 + 事中审批
- 异步工具任务 → 后台任务追踪、等待与取消

QwenPaw 的做法是把这些能力拆到独立组件，再在初始化时“挂回 loop”：

- context manager 通过 hooks 接入： [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L429-L452)
- tool guard 通过 `_acting` 拦截接入： [tool_guard_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L138-L176)

#### 1.1.5 补充：模型自适应重试（从失败中学习“模型怪癖”）

LLM 的失败模式不是纯“网络抖动”，很多时候是模型/供应商对请求格式的隐式约束（例如：某些 thinking 模式模型要求历史里的每条 assistant 消息都携带特定字段）。1.1.5 起，运行时在遇到可识别的错误形态后，会“学习”该模型的约束，并在后续请求里自动修补再试，减少同类错误反复出现。

- 落点：`RetryChatModel` 会在识别到特定 400 错误后注入缺失字段，并缓存为 per-model 能力标记（后续自动应用）  
  - [retry_chat_model.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/providers/retry_chat_model.py#L342-L447)

## 代码走读路线（分）

按“循环的装配顺序”读：

1. 读 `QwenPawAgent.__init__`：哪些组件由外部注入（memory_manager/context_manager/mcp_clients）  
   - [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L99-L215)
2. 读工具注册：如何从配置决定启用、以及 async_execution 的额外任务管理工具注册  
   - [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L216-L333)
3. 读 skills 注册：按 workspace/channel 选择有效 skills  
   - [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L335-L370)
4. 读 MCP 注册：为什么在 agent 构造之后“二次注册 MCP 工具”  
   - [register_mcp_clients](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L477-L541)
5. 读 tool guard 的 `_acting`：如何做 deny/guard/approval 的决策与串行化  
   - [tool_guard_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L138-L176)

## 关键代码讲解（分）

### A) 为什么 ToolGuard 做成 Mixin（MRO 拦截）

把工具守卫放在 Mixin 里有两个工程收益：

- **覆盖面更大**：只要最终工具执行走 `ReActAgent._acting`，就一定会经过 Mixin 的 `_acting`
- **与工具实现解耦**：工具函数保持纯粹（只做能力），治理在统一入口做（更容易审计与改策略）

QwenPaw 在 Mixin 里做的是“决策 + 审批 + 执行委托”：

- 先判定是否需要守卫与审批，再决定是否执行： [tool_guard_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L138-L176)

### B) 工具与 Skills 的分工（能力 vs 任务封装）

一个实用的判断标准：

- **工具（Tool）**：细粒度能力，尽量稳定、可组合、可测试（读文件、搜索、运行 shell）
- **技能（Skill）**：面向任务的“可分发封装”，包含提示词/脚本/依赖/配置（可安装、可启停、可扫描）

QwenPaw 的技能解析与短路体现了这个边界：当用户输入 `/<skill>` 时，Runner 直接把 skill 内容注入最后一条消息，从而让 LLM 以“技能说明书”的方式执行任务，而不需要改 Agent 主循环。  
- [_maybe_inject_skill](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L185-L259)

## 动手练习（可执行）

目标：写出你自己的 Agent 主循环“最小设计说明”（不改代码也能完成）。

1. 写出 5 条 loop 约束（例如：每轮最多 1 次工具调用；工具失败最多重试 1 次；输出必须引用数据来源）。
2. 列出你项目的“工具清单”和“技能清单”，并为每个条目说明为什么是 tool/skill。
3. 选择一个横切关注点（如上下文压缩或工具审批），写出它应该挂在哪个生命周期点，以及你会在代码里用什么机制实现（hook/middleware/mixin/manager）。

### 进阶练习（提升）

1. 以 [react_agent.py:reply](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L1271-L1329) 为参照，写出你自己的“请求上下文注入清单”（至少 5 项：workspace_dir、session_id、预算、超时、观测字段），并标注它们应当如何在调用链中传播。
2. 设计一条“工具失败语义”规范：对每个工具调用，你希望统一包含哪些字段（ok/error_type/retryable/hint/elapsed），以及它如何影响下一轮 reasoning（继续、换工具、升级为人工审批、终止）。

## 验收清单

- [ ] 能解释 ReAct 与 Plan-and-Execute 的工程差异（至少 2 点）
- [ ] 能指出 QwenPawAgent 的四个组装步骤：工具、skills、sys_prompt、hooks
- [ ] 能说明 ToolGuardMixin 的收益与风险（覆盖面/解耦 vs MRO 复杂度）

## 本课小结（总）

Agent 主体的工程化关键，是把模型放进一个受约束的循环，并把“横切关注点”挂到正确的位置。  
在 QwenPaw 中，这套思路体现为：**初始化阶段完成组装**（工具/skills/prompt/hooks），**运行阶段靠拦截与管理器治理**（tool guard、context/memory、MCP manager）。  
下一课我们进入上下文工程、记忆管理与状态管理：它们是让 Agent “长跑不失控”的核心。

## 下一课预告

- 进入 [Module_03_Lesson_02_context_memory_state.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_03_Lesson_02_context_memory_state.md)
