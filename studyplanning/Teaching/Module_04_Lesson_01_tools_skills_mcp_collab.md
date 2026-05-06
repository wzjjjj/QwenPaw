# Module 04 / Lesson 01：工具、Skills、MCP 与多智能体协作（总分总）

## 本课总览（总）

一个可扩展的 Agent 系统，本质上是一个“能力编排平台”：

- **Tools**：细粒度、可组合的能力原语
- **Skills**：面向任务的封装（可分发、可配置、可扫描）
- **MCP**：把外部服务/工具协议化接入（并可热更新）
- **多智能体协作**：把复杂任务拆分为多个角色/能力边界，通过协议进行任务分配与协同

这一课站在项目开发视角，讲清每一层扩展点如何设计接口、如何治理失败模式，并用 QwenPaw 的真实实现做锚点。

## 学习目标

- 能设计一套工具集成框架：注册、调用、选择策略、失败处理、异步任务管理
- 能理解 Skills 的封装与调用逻辑（含 skill-shortcut 注入），并能设计自己的技能库规范
- 能解释 MCP 客户端的生命周期管理与热更新机制，并知道为什么要集中管理
- 能给出一套可落地的多智能体协作模式（任务拆分/路由/仲裁/回收）

## 先修知识

- 已理解 Agent loop 与治理位置（hooks/mixin/manager）
- 已理解上下文/记忆/状态的基本分工

## 本节要回答的关键问题

1. 工具系统的“选择策略”应该放在哪：prompt、agent、runner、还是工具层？
2. Skills 与 Tools 重叠怎么办？怎样避免“技能膨胀”与“工具堆砌”？
3. MCP 为什么要做热更新？热更新如何避免中断正在运行的请求？
4. 多智能体协作如何避免“互相甩锅”和“循环对话”？怎样定义协作契约？

## 核心概念（分）

### 1) 工具管理：集成框架、调用流程、选择策略

工程上一个工具系统至少包括四块：

1. **注册（Registry）**：工具声明与注册，含开关与配置  
2. **调用（Invocation）**：结构化参数、执行、返回、错误语义  
3. **选择（Selection）**：什么时候用哪个工具（策略可以来自模型/规则/打分/历史）  
4. **治理（Governance）**：权限、审批、审计、资源预算

QwenPaw 的工具注册集中在 Agent 初始化阶段，并支持从配置中决定启用与 async_execution：  
- [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L216-L333)

其中一个关键工程点是：如果有任意工具启用了 async_execution，则自动注册后台任务管理工具（view/wait/cancel）供模型管理任务生命周期：  
- [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L303-L332)

这体现了“工具调用流程”的完整性：**发起 → 追踪 → 等待/取消 → 回收**。

### 2) Skills：技能库设计、封装与调用逻辑

把 Skills 当作“可安装的任务包”，你需要定义三层规范：

- **目录规范**：一个 skill 就是一个目录（运行时 identity=目录名）
- **描述规范**：SKILL.md 的 frontmatter + 正文（供 UI 展示与 LLM 注入）
- **安全规范**：安装/导入前必须扫描（提示词注入/命令注入/硬编码密钥等）

QwenPaw 在 skills_manager 中明确了“skills 不是随意字符串”，而是带版本、语言、来源、锁与扫描的受控资源：  
- [skills_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/skills_manager.py#L27-L60)（技能扫描依赖与通道路由枚举）

调用逻辑上，QwenPaw 采用“skill shortcut”策略：当用户输入 `/<skill>` 时，Runner 可直接返回技能信息或把 skill 内容注入最后一条用户消息（短路/重写），让 LLM 按技能说明执行任务：  
- [_maybe_inject_skill](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L185-L259)

这个设计的好处：

- **不污染 Agent 主循环**：新增 Skill 不需要改 agent loop
- **技能可热更新**：因为 agent 每次 query 都会重新创建（Runner 逻辑里已说明）
- **交互体验好**：`/skill` 无输入时直接显示技能信息（更像 CLI）

### 3) MCP：多智能体协作协议与外部能力协议化接入

把 MCP 当作“外部能力的标准化接口”，它主要解决：

- 外部工具/服务的统一发现与调用
- 生命周期管理（连接、重连、关闭）
- 热更新（配置变更后替换 client，不重启进程）

QwenPaw 把 MCP 客户端管理集中到 `MCPClientManager`，核心设计点是“锁只保护 swap，不保护慢连接”：  
- [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/mcp/manager.py#L90-L132)

这能减少阻塞，并支持运行时替换旧 client。

### 4) 多智能体协作：通信机制、任务分配与协作模式

多智能体协作不等于“多个 LLM 聊天”，而是把系统拆成多个 **责任边界清晰** 的角色：

- 专家型 agent：领域内推理与产出
- 执行型 agent：工具密集调用与文件操作
- 审计型/守卫型 agent：检查风险、给出批准/拒绝理由（或生成审批提示）

QwenPaw 在系统层面提供了多 agent 的运行时隔离与生命周期：  
- [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)

协作的关键是“契约”，建议你在项目中明确：

- 任务分解契约：输入、输出、可调用工具范围
- 失败语义：失败类型、可重试/不可重试、回退策略
- 资源预算：token、工具调用次数、时间

## 代码走读路线（分）

1. 工具注册与 async_execution 的任务管理工具：  
   - [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L216-L333)
2. skills 的安全扫描与 API 层错误返回形态（结构化 findings）：  
   - [skills.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/skills.py#L70-L111)
3. runner 的 skill shortcut 注入：  
   - [runner.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L185-L259)
4. MCP client manager 的 replace_client：  
   - [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/mcp/manager.py#L90-L132)
5. 多 agent workspace 的懒加载与并发去重：  
   - [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)

## 动手练习（可执行）

1. 设计一个新工具（不写代码也可）：给出 name、参数 schema、返回结构、失败语义（可重试/不可重试）。
2. 设计一个新 Skill：写出目录结构、SKILL.md 的 frontmatter 字段、以及它依赖哪些工具。
3. 设计一个 MCP client 配置（stdio 或 http）：列出你希望它暴露的工具列表与权限策略。
4. 设计一个“多智能体协作流程”：至少包含 2 个角色 agent，定义它们的输入输出与边界。

## 验收清单

- [ ] 能解释 Tools vs Skills vs MCP 的分工（并能举出你项目的例子）
- [ ] 能描述 QwenPaw 的 skill shortcut 注入机制（为何适合热更新）
- [ ] 能指出 MCPClientManager 的热更新关键点（connect 在锁外、swap 在锁内）
- [ ] 能写出一个多智能体协作契约（输入/输出/失败语义/预算）

## 本课小结（总）

扩展能力的关键不是“加功能”，而是“加扩展点”。  
在 QwenPaw 中，工具注册、skills 分发、MCP 生命周期管理、多 agent runtime 各自有明确边界，并通过 runner/agent 的装配把它们组合成系统能力。  
下一课我们进入安全与治理：让这些扩展点在真实环境中可控可审计。

## 下一课预告

- 进入 [Module_05_Lesson_01_security_and_governance.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_05_Lesson_01_security_and_governance.md)

