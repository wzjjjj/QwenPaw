# 课程大纲：Agent 后端工程（以 QwenPaw 为例）

本大纲只放“课程结构 + 每课学习目标”。正文请进入各 Lesson 文件。

## 总体目标

- 能从零设计并实现一个可运行、可扩展、可治理、可运维的 Agent 后端系统（而不只是“把 LLM 接上工具”）
- 能读懂并改造 QwenPaw 的关键链路（入口/路由/runner/agent/渠道/治理），把设计方法迁移到自己的项目
- 能以“高级后端大模型工程师”的标准思考并落地：接口契约、状态机、失败语义、并发与资源预算、可观测性与测试、上线与迭代

## 课程毕业要求（你学完应当能做到）

- 能画出并解释一张完整架构图（入口/路由/runner/agent/tools/skills/mcp/context&memory/security/channels/observability）
- 能独立完成一次“能力扩展”并配套验证：新增一个 tool 或 skill（或一个最小 channel），包含失败语义、守卫/审批策略、以及最小测试
- 能定位并排障 3 类常见线上问题：上下文爆炸/输出失控、工具卡死/超时、渠道乱序/并发竞态（给出日志与状态机层面的证据）

## 练习分级（每课至少包含其一）

- 基础（必做）：读链路 + 画图/写契约/跑通命令
- 进阶（提升）：小改动 + 可验证（加日志/加测试/加守卫规则/调优阈值）
- 挑战（选做）：端到端能力（新工具/新技能/新渠道/新协作模式）+ 回归验证

## 模块结构（总分总）

### Module 01：总体概述（先建立全局坐标）

- Lesson 01：行业现状、应用前景与学习路径  
  - 理解 Agent 技术从“对话”走向“可执行系统”的关键驱动力  
  - 给出本课程的学习框架、成果验收与关键技术节点  
  - 建立 QwenPaw 的主链路心智模型（入口 → runner → agent → tools/skills → memory/context → security）
  - 练习产出：画出“单次请求主链路”时序图 + 标出 4 个关键入口文件
  - 入口：[Module_01_Lesson_01_overview.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_01_Lesson_01_overview.md)

- Lesson 02：入口与数据形状（Entrypoints & Shapes）  
  - 能列出 QwenPaw 的 4 类入口（CLI / Console API / Runtime API / Channels）及其职责边界  
  - 能整理一次请求的最小字段集（agent_id/session_id/root_session_id/channel/user_id/input blocks）并标注来源与使用位置  
  - 练习产出：入口矩阵 + shape 字段表（带代码锚点证据）
  - 入口：[Module_01_Lesson_02_entrypoints_and_shapes.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_01_Lesson_02_entrypoints_and_shapes.md)

- Lesson 03：配置与会话持久化（Config & Sessions）  
  - 理解配置加载的工程约束：mtime 缓存、auto-repair、校验失败降级、回退默认  
  - 理解会话持久化的工程约束：跨平台文件名、JSON 损坏恢复、最坏情况下如何继续运行  
  - 练习产出：配置失败语义表 + 会话失败语义表 + 最小不变量清单
  - 入口：[Module_01_Lesson_03_config_and_sessions.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_01_Lesson_03_config_and_sessions.md)

### Module 02：项目定位与整体架构（把“要做什么”变成“系统怎么落地”）

- Lesson 01：项目定位 + 架构分层 + 模块交互  
  - 能给自己的 Agent 项目写出场景/价值主张/目标用户画像  
  - 能画出分层架构（API 路由层 / Runner 层 / Agent 层 / 能力层 / 基础设施与治理层）  
  - 能在代码里找到 QwenPaw 的层间边界与交互点（尤其是 multi-agent 与 agent-scoped 路由）
  - 练习产出：输出“定位 → 架构”一页纸（含边界、风险面、扩展点、治理点）
  - 入口：[Module_02_Lesson_01_positioning_and_architecture.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_02_Lesson_01_positioning_and_architecture.md)

### Module 03：Agent 主体与上下文/记忆/状态（把“不确定的推理”装进“可控的执行循环”）

- Lesson 01：Agent 主体（决策系统 / 推理机制 / 行为模式）  
  - 理解 ReAct loop 的工程实现：输入组织、工具注册、sys_prompt、hooks  
  - 能解释 QwenPawAgent 初始化时各组件的责任边界  
  - 练习产出：写出你的 agent loop 最小设计说明（输入/状态/约束/输出 + 失败语义）
  - 入口：[Module_03_Lesson_01_agent_core_loop.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_03_Lesson_01_agent_core_loop.md)

- Lesson 02：上下文工程 + 记忆管理 + 状态管理  
  - 理解“上下文窗口”和“长期记忆”在工程上的分工：压缩/裁剪 vs 检索/沉淀  
  - 能描述并实现一个最小可用的上下文更新策略与记忆检索策略  
  - 能在系统里识别关键状态（会话/任务/计划/工具审批）并建模为可观测状态机  
  - 练习产出：输出 1 张状态机表 + 1 份上下文/记忆策略（含阈值与触发时机）
  - 入口：[Module_03_Lesson_02_context_memory_state.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_03_Lesson_02_context_memory_state.md)

### Module 04：工具、Skills、MCP 与多智能体协作（扩展与编排）

- Lesson 01：工具管理 + Skills + MCP + 多智能体协作  
  - 能设计工具接口、调用流程与选择策略（含失败模式）  
  - 能理解 Skills 的“封装形态”和“运行时加载方式”  
  - 能解释 MCP 客户端的生命周期管理与热更新策略  
  - 能给出一套可落地的多智能体协作模式（主从/团队/路由/仲裁）  
  - 练习产出：设计 1 个 tool + 1 个 skill + 1 个 MCP 配置草案（带失败语义与权限策略）
  - 入口：[Module_04_Lesson_01_tools_skills_mcp_collab.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_04_Lesson_01_tools_skills_mcp_collab.md)
- Lesson 02：多智能体协作的工程落地（机制 + 模式 + 反模式）  
  - 能区分“运行时多智能体（workspace 隔离）”与“协作多智能体（agent-to-agent 通信）”  
  - 能走通 inter-agent chat：list_agents/chat_with_agent/submit_to_agent/check_agent_task  
  - 能掌握 session_id/root_session_id 在跨 agent 协作里的作用与风险点  
  - 练习产出：跑通一次前台协作 + 一次后台委派，并画出 inter-agent 时序图
  - 入口：[Module_04_Lesson_02_multi_agent_collaboration_patterns.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_04_Lesson_02_multi_agent_collaboration_patterns.md)

### Module 05：安全与治理（让 Agent 可在真实环境运行）

- Lesson 01：安全与治理（数据安全 / 权限控制 / 工具守卫 / 技能扫描）  
  - 理解“事前守卫 + 事中审批 + 事后审计”的分层治理方法  
  - 能把风险识别与审批结果回灌到对话上下文中，让 LLM 自己学会规避  
  - 练习产出：输出风险地图 + 设计一份 guard scope/deny list + 最小扫描规则集
  - 入口：[Module_05_Lesson_01_security_and_governance.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_05_Lesson_01_security_and_governance.md)

### Module 06：API 路由层（系统边界与接口契约）

- Lesson 01：API 路由层设计  
  - 能组织路由模块、划分资源边界与版本/多租户策略  
  - 能理解 QwenPaw 的 agent-scoped 路由与上下文注入机制  
  - 能为 “agent + workspace + skills/tools/mcp” 设计稳定 API 契约  
  - 练习产出：设计 2 个接口 schema + 1 个长任务状态接口 + 1 个审批流接口
  - 入口：[Module_06_Lesson_01_api_routing_layer.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_06_Lesson_01_api_routing_layer.md)

### Module 07：综合实践与总结（回到“总”）

- Lesson 01：典型案例实战 + 维度协同 + 评估与进阶  
  - 完成一个端到端的案例：从需求 → 架构 → 扩展点（Skills/Tools/MCP）→ 治理 → 验收  
  - 学会系统优化方法（性能、可靠性、可扩展性、安全）  
  - 给出学习成果评估与进阶路线（读源码/改需求/写测试/做发布）  
  - 练习产出：完成一个“毕业作品”变体（最小可运行 + 可验证 + 可治理）
  - 入口：[Module_07_Lesson_01_capstone_and_summary.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_07_Lesson_01_capstone_and_summary.md)

### Module 08：Channels（渠道）与异步链路（把 Agent 接入真实世界）

- Lesson 01：Channels 的异步心智模型（读懂线程、事件循环与队列）  
  - 能解释 event loop / coroutine / task / async generator 在渠道链路中的角色  
  - 能理解 weixin 为何采用“后台线程 + 自建 event loop”，以及 thread-safe 入队的必要性  
  - 练习产出：指出入队与串行消费的关键位置，并能解释 3 类并发 bug
  - 入口：[Module_08_Lesson_01_async_mentality_for_channels.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_08_Lesson_01_async_mentality_for_channels.md)
- Lesson 02：Weixin 一条消息的端到端链路（入参/出参/调用方逐节点串联）  
  - 能把平台消息 → native payload → AgentRequest → runner event → 回包的主链路串起来  
  - 能对每个关键节点说清调用方、入参形状、出参/副作用（以 weixin 为主例）  
  - 练习产出：能定位 meta 丢失/回包失败的排障节点，并给出日志落点
  - 入口：[Module_08_Lesson_02_weixin_inbound_to_reply_end_to_end.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_08_Lesson_02_weixin_inbound_to_reply_end_to_end.md)
- Lesson 03：Console 对比 + 新渠道设计清单 + 如何验证  
  - 能对比 Console/Weixin 的模型差异，理解 BaseChannel/ChannelManager 的抽象边界  
  - 能按“必做/选做”清单设计并接入一个新渠道，并知道如何用现有测试/契约做验证  
  - 练习产出：写出一个新渠道的契约草图 + 最小验证用例清单
  - 入口：[Module_08_Lesson_03_console_contrast_extension_and_tests.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_08_Lesson_03_console_contrast_extension_and_tests.md)

### Module 09：部署接口三件套（模型侧 / 工具侧 / 安全侧）

- Lesson 01：部署接口三件套：模型侧 / 工具侧 / 安全侧（算法工程师视角）  
  - 说清楚企业部署里必须对齐的三块接口面：Provider（推理服务）、Tool/MCP（能力入口）、Tool Guard/Approval（治理入口）  
  - 练习产出：为一个 agent 写出“最小可用的部署清单”（模型/工具/审批默认策略）
  - 入口：[Module_09_Lesson_01_deployment_interfaces_model_tools_security.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_09_Lesson_01_deployment_interfaces_model_tools_security.md)

### Module 10：Python 工程能力专修（面向 Agent/后端真实代码）

- Lesson 01：asyncio 工程化：并发、线程与取消（以 QwenPaw 为例）  
  - 能读懂项目里的异步边界：哪些是 `async` I/O，哪些必须 `to_thread`，哪些需要队列串行  
  - 能解释：为什么会出现“线程 + 自建 event loop”，以及如何安全地停机与清理任务  
  - 练习产出：给 1 条异步链路补齐“超时 + 取消 + 资源释放”的最小闭环（可用测试验证）
  - 入口：[Module_10_Lesson_01_asyncio_in_agent_systems.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_10_Lesson_01_asyncio_in_agent_systems.md)
- Lesson 02：类型注解的工程落地：把复杂系统变得可读可改  
  - 能在项目里熟练使用 `| None`、`TypedDict`、`TYPE_CHECKING`、泛型容器等常见模式  
  - 能用类型系统表达接口契约（例如 tool 输出、channel payload、provider/model 信息）  
  - 练习产出：为一个真实模块补齐类型边界（函数签名 + 数据结构），并通过最小单测或运行路径验证
  - 入口：[Module_10_Lesson_02_typing_for_large_codebases.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_10_Lesson_02_typing_for_large_codebases.md)
- Lesson 03：Pydantic v2：配置模型与 Schema 是怎么“在工程里跑起来”的  
  - 能手写/维护 BaseModel：Field、default_factory、model_config、validator（v2）  
  - 能解释“配置加载/校验/落盘/回显”链路，并定位校验失败原因  
  - 练习产出：新增 1 个配置字段（带校验）并让它从 config.json/接口进入系统，且有最小测试
  - 入口：[Module_10_Lesson_03_pydantic_v2_in_qwenpaw.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_10_Lesson_03_pydantic_v2_in_qwenpaw.md)
- Lesson 04：装饰器与上下文管理器：理解框架式代码的“隐式控制流”  
  - 能看懂常见装饰器：click 命令栈、Pydantic validator、缓存（lru_cache）与权限/治理式包装  
  - 能读懂/编写 context manager：资源申请与释放、异常吞吐策略（suppress）、可重入与幂等  
  - 练习产出：把 1 个“手写 try/finally 清理块”重构成可复用的 context manager，并补齐测试
  - 入口：[Module_10_Lesson_04_decorators_and_context_managers.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_10_Lesson_04_decorators_and_context_managers.md)
- Lesson 05：测试与调试：让你能“大胆改、快速回归、不怕线上”  
  - 能在 QwenPaw 的 pytest 体系里写异步测试（pytest-asyncio）、隔离临时目录、mock 外部依赖  
  - 能把日志/异常当成接口：用可观测性证据定位并修复 bug（超时、竞态、配置错误、序列化问题）  
  - 练习产出：为一次真实改动补齐回归用例（unit + 最小集成），并能解释覆盖的失败模式
  - 入口：[Module_10_Lesson_05_testing_debugging_and_tooling.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_10_Lesson_05_testing_debugging_and_tooling.md)

## Next Iteration Backlog（迭代式推进的候选课）

以下是对齐 nanobot 风格（Harness Engineering 链路拆解）的候选增补课，按你反馈逐次推进：

- Module 02（Turn/Queue）
  - `Module_02_Lesson_01_messagebus_and_queue_semantics.md`（TODO）
  - `Module_02_Lesson_02_runner_turn_state_machine.md`（TODO）
  - `Module_02_Lesson_03_streaming_artifacts_and_events.md`（TODO）
- Module 03（Runner 可靠性）
  - `Module_03_Lesson_01_runner_loop_skeleton.md`（TODO）
  - `Module_03_Lesson_02_injection_and_checkpoint.md`（TODO）
  - `Module_03_Lesson_03_errors_and_recovery.md`（TODO）
- Week 学习计划（TODO）：`Week_01.md` … `Week_08.md`
- Capstone（TODO）：`Capstone_Project_Brief.md`
