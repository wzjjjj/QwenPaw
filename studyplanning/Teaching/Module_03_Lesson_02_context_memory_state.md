# Module 03 / Lesson 02：上下文工程 + 记忆管理 + 状态管理（总分总）

## 本课总览（总）

Agent 项目里，“能回答”只是起点，“能长期稳定运行”才是难点。  
你需要同时管理三件事：

- **上下文（Context Window）**：模型每次推理能看到的内容（短期、贵、易爆炸）
- **记忆（Memory）**：跨会话/长期的沉淀与检索（长期、慢、需隐私边界）
- **状态（State）**：系统在运行时的可观测状态机（计划/工具审批/任务/会话/重载）

这一课用 QwenPaw 的 `BaseContextManager`、`BaseMemoryManager` 与工具守卫流程作为锚点，讲清“采集→处理→表示→更新”的工程策略，并给出可落地的实现与验收方法。

## 学习目标

- 能区分上下文与记忆的工程分工，并为自己的系统选定最小可用策略
- 能解释“上下文压缩”“工具结果裁剪”“记忆检索/沉淀”的典型实现点
- 能把关键运行态建模为状态机，并给出监控/排障的落点

## 先修知识

- 已理解 Agent 主循环的装配点（hooks/mixin/manager）
- 能读懂抽象基类与接口定义

## 本节要回答的关键问题

1. 为什么“上下文越塞越多”会导致性能下降与幻觉上升？工程上如何治理？
2. 记忆系统的“写入/检索/更新”分别应当在什么时候触发？
3. 状态管理在 Agent 项目中到底管理什么？哪些状态必须可观测？
4. QwenPaw 为什么用 ContextManager 的 hook 来做压缩/裁剪？为什么 MemoryManager 用工具暴露给 agent？

## 核心概念（分）

### 1) 上下文工程：采集、处理、表示、动态更新

把“上下文工程”拆成四步，工程上就能落地：

1. **采集（Collect）**：收集会影响决策的信息源  
   - 用户输入、系统提示、历史对话、工具结果、环境上下文（时间/渠道/agent_id）
2. **处理（Process）**：清洗、裁剪、结构化、去噪  
   - 典型策略：剔除无用 tool 输出、将长日志变成摘要、把多条消息合并为结构化事实
3. **表示（Represent）**：让模型“更容易用”  
   - 常见形态：系统提示中的规则、摘要段落、结构化 JSON（需考虑模型兼容）
4. **更新（Update）**：何时压缩、何时丢弃、何时提取长期记忆  
   - 关键是策略：触发阈值、优先级、过期机制、任务切换策略

QwenPaw 在接口层明确了 ContextManager 的职责：压缩、裁剪、健康检查与生命周期 hook：  
- [base_context_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/base_context_manager.py#L15-L175)

并在 Agent 初始化时把这些 hook 挂回 loop：  
- [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L429-L452)

你可以把它理解为：**ContextManager 不是“存储”，而是“上下文窗口治理器”。**

#### 1.1.5 补充：上下文压缩的降级机制（压缩失败/被禁用时不“中止流程”）

1.1.5 之后，上下文治理的默认策略更偏“保可用”：当 LLM 压缩失败（异常、输出不合规、空摘要等）或压缩被禁用时，不再直接中止或维持原分割，而是会放宽保留窗口重新切分，尽可能把更多“近期轮次”留在上下文里。

- 目标：优先保留最近消息，宁可丢更老的历史，也不要让压缩流程卡死或让上下文治理失效
- 落点：`LightContextManager.pre_reasoning` 内对 compaction 结果的兜底分支  
  - [light_context_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L710-L921)

### 2) 记忆管理：短期 vs 长期，存储结构、检索与更新

一个工程化的记忆系统至少要回答三件事：

- **写什么（Write Policy）**：哪些信息值得沉淀（偏好、长期目标、稳定事实、项目约束）  
- **怎么取（Retrieve Policy）**：按关键词/语义/标签/时间检索？召回多少？如何去噪？  
- **怎么更新（Update Policy）**：冲突信息如何处理？过期如何淘汰？如何压缩/合并？

QwenPaw 用抽象基类把记忆管理器定义成“可注入的后端”，并支持后台摘要任务队列：  
- [base_memory_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/base_memory_manager.py#L18-L116)（接口与可选 summarize/retrieve/dream）  
- [base_memory_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/base_memory_manager.py#L130-L182)（后台 summarize worker 与队列）

更关键的是：它把部分记忆能力以工具形式暴露给 agent（list_memory_tools），这意味着你可以让 LLM 自己在需要时检索/写入，但仍在工具层受治理。  
- [BaseMemoryManager.list_memory_tools](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/base_memory_manager.py#L66-L76)

#### 1.1.5 补充：记忆检索支持 CJK 字符级分词（保留拉丁字母/数字连续串）

当你的用户 query 含中文/日文/韩文且没有明显空格分词时，纯“按空格切词”会导致检索召回很差。1.1.5 起，记忆搜索会对 CJK 以“字符 1-gram”分词，同时保留拉丁字母和数字的完整连续串，兼顾中文召回与英文/ID 精确性。

- 落点：`ReMeLightMemoryManager.tokenize_query` 在 search 前对 query 做 tokenization  
  - [reme_light_memory_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/reme_light_memory_manager.py#L279-L388)

### 3) 状态管理：把“隐式行为”显式化为状态机

Agent 项目里的状态不止是“对话历史”。典型关键状态包括：

- **会话状态**：当前 session_id、消息序列、是否在流式输出
- **执行状态**：当前是否在一次推理/一次工具调用/一次回复阶段
- **工具审批状态**：待审批、已拒绝、超时、已放行
- **异步任务状态**：后台工具任务（view/wait/cancel）、mission/plan 的阶段状态
- **运行时生命周期状态**：workspace 启动中、可服务、重载中、旧实例清理中

QwenPaw 的工具守卫流程把“审批状态”落在了 tool 执行前的统一入口 `_acting`，并通过锁避免并发状态竞态：  
- [tool_guard_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L138-L176)

QwenPaw 的多智能体管理把“workspace 生命周期状态”显式化为：缓存、pending_start 去重、延迟清理：  
- [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)（懒加载 + 并发去重）  
- [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L138-L216)（零停机重载的旧实例清理策略）

## 代码走读路线（分）

1. 读 ContextManager 接口：hook 与 compact_context 是怎样被定位成“治理行为”的  
   - [base_context_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/base_context_manager.py#L82-L223)
2. 读 Agent 注册 context hooks：每个 hook 处于 loop 的哪个阶段  
   - [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L429-L452)
3. 读 MemoryManager 接口：为什么 summarize/retrieve/dream 是 optional  
   - [base_memory_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/base_memory_manager.py#L78-L128)
4. 读 ToolGuardMixin 的状态串行化：为什么“决策”要加锁而“执行”放锁外  
   - [tool_guard_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L149-L176)

## 实践：给你自己的项目选一套“最小可用策略”（分）

### A) 上下文策略（MVP）

- 触发阈值：当 token 或消息条数超过阈值，压缩旧消息为摘要
- 工具结果策略：工具输出超过阈值就截断 + 生成结构化摘要（保留关键字段）
- 任务切换：当用户显式切换任务（或 /new），把当前摘要固化，清空短期窗口

### B) 记忆策略（MVP）

- 写入：只写入“稳定偏好/长期事实/项目约束”，避免写入一次性信息
- 检索：每次推理前，用最近用户问题做 query，召回 3-5 条，拼接为短段落
- 更新：若新信息与旧记忆冲突，新增“覆盖记录”并标注时间/来源（不要原地覆盖）

### C) 状态策略（MVP）

- 定义 5 个关键状态：Idle、Reasoning、Acting、AwaitingApproval、Replying
- 为每个状态定义：进入条件、退出条件、允许的操作（例如 AwaitingApproval 不允许继续 Acting）
- 每次状态迁移写一条结构化日志（便于复盘与压测）

## 动手练习（可执行）

1. 写一个“状态机表格”（用 Markdown 表即可），覆盖上述 5 状态与迁移条件。
2. 列出你系统中“上下文来源清单”，并给每项打分：重要性/稳定性/隐私风险。
3. 选一个工具输出（例如 grep 的结果），设计一份“裁剪后摘要结构”（字段：query、top_matches、files、notes）。

## 验收清单

- [ ] 能清晰区分：上下文窗口治理 vs 长期记忆沉淀（职责不混）
- [ ] 有一份最小可用的上下文/记忆/状态策略（能写到配置或文档中）
- [ ] 能指出 QwenPaw 的三个落点：context hooks、memory manager 接口、tool guard 的审批入口

## 本课小结（总）

上下文、记忆、状态是 Agent 长期运行的三根支柱：  
上下文决定每次推理“看见什么”；记忆决定系统“长期学到什么”；状态决定系统“什么时候做什么、出了事怎么查”。  
下一课我们转向扩展：工具、Skills、MCP 与多智能体协作，建立“可插拔能力”与“协作编排”的方法。

## 下一课预告

- 进入 [Module_04_Lesson_01_tools_skills_mcp_collab.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_04_Lesson_01_tools_skills_mcp_collab.md)
