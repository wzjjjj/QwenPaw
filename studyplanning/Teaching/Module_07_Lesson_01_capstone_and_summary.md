# Module 07 / Lesson 01：综合实践与总结（总分总）

## 本课总览（总）

到这里，你已经分别学习了定位、架构、agent loop、上下文/记忆/状态、工具/skills/MCP、多智能体协作、安全治理、API 路由层。  
这节课把这些维度“重新编排”为一个可交付的端到端案例，并给出优化方法与进阶路线，让你能把知识迁移到自己的 Agent 项目。

## 学习目标

- 能完成一个典型 Agent 项目案例的端到端设计：需求 → 分层架构 → 扩展点 → 治理 → API 契约 → 验收
- 能给出核心维度的协同优化方法（性能、可靠性、可扩展性、安全）
- 能使用可验证的标准评估学习成果，并制定下一阶段学习路线

## 先修知识

- 已完成前 6 个模块

## 典型案例：个人研究助手（可迁移到企业/研发场景）

### 需求（用工程语言表达）

目标：做一个“研究助手 Agent”，输入一个研究问题，输出可追溯的调研结论，并能在本地工作目录沉淀资料与摘要。

功能需求：

- 支持文件检索与摘要（本地文档）
- 支持网页检索（可选：通过 MCP 或内置浏览器/搜索工具）
- 支持将结论沉淀为长期记忆（用户偏好/项目约束/关键事实）
- 支持安全治理：限制敏感路径访问、危险 shell、技能扫描
- 支持多智能体：Researcher（检索/阅读）、Writer（结构化输出）、Auditor（安全与一致性检查）

非功能需求：

- 可观测：能看见当前任务状态、工具审批状态、失败原因
- 可扩展：后续可新增“论文下载/引用管理/知识库”

### 架构（用分层表达）

建议分层（复用 Module 02）：

- API 路由层：chats、skills、tools、mcp、workspace、plan
- Runner 层：处理 `/skill` 短路、流式输出、错误兜底
- Workspace & Multi-Agent：隔离多个 agent、支持零停机重载
- Agent 层：组装工具/skills/context/memory/security
- 能力层：工具、skills、mcp
- 治理层：工具守卫、技能扫描、审计日志

在 QwenPaw 的落点参考：

- 动态 runner（按 agent_id 选择 workspace）：[_app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L126)
- 多 agent workspace 管理： [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)
- Agent 组装： [react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L99-L215)

### 扩展点设计（Tools / Skills / MCP）

1. **Tools**（细粒度能力）
   - read_file / grep_search / glob_search：本地资料检索  
   - browser_use 或 MCP：网页检索/抓取  
2. **Skills**（任务封装）
   - “research_summary” skill：规定输出结构、引用格式、写入文件策略
   - “citation_check” skill：对引用一致性与来源进行检查
3. **MCP**（外部接入）
   - 连接一个“论文检索服务”或“向量库服务”作为 MCP client（可热更新）
   - MCP client 生命周期集中管理： [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/mcp/manager.py#L23-L132)

### 上下文/记忆/状态协作策略（让系统可控）

- 上下文：只保留当前问题、关键证据、工具摘要；长日志裁剪
- 记忆：沉淀用户偏好（引用格式、输出语言、关注领域）与项目约束（禁止访问路径/保密要求）
- 状态：任务状态机（planning/researching/writing/auditing/done），审批状态机（pending/approved/denied/timeout）

接口锚点：

- ContextManager 接口： [base_context_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/base_context_manager.py#L15-L223)
- MemoryManager 接口： [base_memory_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/base_memory_manager.py#L18-L116)

### 安全与治理（必须贯穿全链路）

建议治理策略：

- 工具守卫默认开启，危险工具 deny，敏感工具必须审批
- 技能导入/安装必须扫描，失败返回结构化 findings

QwenPaw 参考锚点：

- 工具守卫执行入口： [tool_guard_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L138-L176)
- 守卫引擎与默认 guardians： [engine.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/security/tool_guard/engine.py#L54-L235)
- skills 扫描失败结构化返回： [skills.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/skills.py#L70-L111)

## 系统优化方法（分）

### 1) 性能优化

- 控制上下文成本：压缩/裁剪 tool 输出，减少无效 tokens
- 并发与锁策略：像 MCP manager 一样，慢操作在锁外，swap 在锁内
- 异步工具：启用后台任务管理工具（view/wait/cancel），避免阻塞 loop

### 2) 可靠性优化

- 明确失败语义：哪些错误可重试（网络/超时），哪些必须停止（权限/安全拒绝）
- 顶层兜底：runner 捕获异常并输出结构化错误（便于前端与模型处理）
- 零停机重载：旧 workspace 延迟清理，避免中断在途任务

### 3) 可扩展性优化

- 稳定扩展点：工具注册表、skills 目录规范、MCP client manager
- 把横切关注点组件化：context/memory/security 通过 hook/mixin 接入
- 把“策略”配置化：guard scope、approval level、skills enable、channel routing

### 4) 安全优化

- 守卫前移：工具执行前统一拦截（而不是工具内部零散检查）
- 审批可解释：返回 findings 与理由，让用户能判断、让模型能学习
- 技能扫描常态化：把风险挡在“安装时”，而不是运行时救火

## 动手练习（毕业作品：从“能跑”到“可交付”）

你可以选一个你更感兴趣的方向作为毕业作品，要求只有三点：**可跑、可验证、可治理**。

### 练习 A（必做）：用 QwenPaw 的“多 agent + workspace”搭一个研究助手雏形

1. 创建 3 个 agent（Researcher / Writer / Auditor），并让它们在工具权限上体现责任边界（例如：Auditor 禁止执行 shell；Writer 禁止读敏感路径）。
2. 为主控 agent 写一份最小“协作契约”（输入/输出/失败语义/预算），并放到 workspace 的提示文件里（AGENTS.md 或 PROFILE.md）。
3. 走通一次协作链路：主控把“网页/文档证据收集”交给 Researcher，再让 Writer 产出结构化结论，最后让 Auditor 做一致性与风险复核。

### 练习 B（提升）：做一次“可观测”改造（只加最小必要信息）

目标是让你排障时不依赖“猜”，而依赖证据。

1. 给“关键状态”选一套统一字段（至少包含：agent_id、session_id、root_session_id、phase、tool_name、status、elapsed_ms）。
2. 在你认为最关键的 2 个位置设计日志/事件点（例如：runner 进入/退出、工具审批决策、渠道入队/消费）。
3. 写下你会如何用这些字段定位 3 类问题：工具卡死/超时、回复乱序、上下文爆炸。

### 练习 C（挑战）：做一次“可验证”的能力扩展

从下面任选其一：

1. 新增一个 tool（偏后端工程）：定义参数 schema、返回结构、失败语义（可重试/不可重试/需要审批），并写一个最小单测验证其边界条件。
2. 新增一个 skill（偏产品化封装）：把一个复杂任务封装成可安装目录（SKILL.md + 运行说明），并验证启用/禁用对运行期行为的影响。
3. 新增一个最小 channel（偏异步链路）：复用 BaseChannel 契约完成 native → AgentRequest 与 send，写下最小验证清单（构造、入站、出站、串行）。

## 学习成果评估（验收清单）

- [ ] 输出一份你自己的 Agent 项目一页纸：定位、分层架构、扩展点、安全策略、API 契约
- [ ] 能指出 QwenPaw 中每个维度对应的实现入口（至少到文件级）
- [ ] 能完成一个最小能力扩展：新增一个 tool 或 skill，并说明失败语义与治理点
- [ ] 有一份状态机定义：任务状态 + 审批状态（至少 5 个状态）

## 进阶学习建议（回到“总”）

1. 从“读源码”走向“改需求”：选一个维度（例如工具治理），尝试做一个可测试的小改动
2. 从“单体”走向“协作”：设计一个多智能体协作契约，让不同 agent 的边界更清晰
3. 从“能跑”走向“可运维”：补齐可观测性（日志、指标、错误结构化输出）
4. 从“工具堆叠”走向“平台化”：把能力沉淀成可安装 skills，把外部服务协议化为 MCP

## 课程总结（总）

Agent 项目开发的核心，不是“让模型更聪明”，而是把不确定性装进可控的系统：  
用分层架构界定边界，用 runner/agent loop 组织执行，用上下文/记忆/状态支撑长期运行，用工具/skills/MCP 提供扩展能力，用安全治理保证可在真实环境上线。  
如果你愿意继续深入，可以从你最关心的维度切入，把 QwenPaw 当成一套成熟的“工程样板”，逐步迁移到自己的系统中。
