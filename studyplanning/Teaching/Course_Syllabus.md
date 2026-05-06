# 课程大纲：Agent 项目开发（以 QwenPaw 为例）

本大纲只放“课程结构 + 每课学习目标”。正文请进入各 Lesson 文件。

## 总体目标

- 能从零设计一个可运行、可扩展、可治理的 Agent 系统（而不只是“把 LLM 接上工具”）
- 能读懂 QwenPaw 的关键链路，并把其设计方法迁移到自己的项目
- 能基于明确的接口契约与安全边界扩展：工具、Skills、MCP、多智能体、API 路由

## 模块结构（总分总）

### Module 01：总体概述（先建立全局坐标）

- Lesson 01：行业现状、应用前景与学习路径  
  - 理解 Agent 技术从“对话”走向“可执行系统”的关键驱动力  
  - 给出本课程的学习框架、成果验收与关键技术节点  
  - 建立 QwenPaw 的主链路心智模型（入口 → runner → agent → tools/skills → memory/context → security）
  - 入口：[Module_01_Lesson_01_overview.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_01_Lesson_01_overview.md)

### Module 02：项目定位与整体架构（把“要做什么”变成“系统怎么落地”）

- Lesson 01：项目定位 + 架构分层 + 模块交互  
  - 能给自己的 Agent 项目写出场景/价值主张/目标用户画像  
  - 能画出分层架构（API 路由层 / Runner 层 / Agent 层 / 能力层 / 基础设施与治理层）  
  - 能在代码里找到 QwenPaw 的层间边界与交互点（尤其是 multi-agent 与 agent-scoped 路由）
  - 入口：[Module_02_Lesson_01_positioning_and_architecture.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_02_Lesson_01_positioning_and_architecture.md)

### Module 03：Agent 主体与上下文/记忆/状态（把“不确定的推理”装进“可控的执行循环”）

- Lesson 01：Agent 主体（决策系统 / 推理机制 / 行为模式）  
  - 理解 ReAct loop 的工程实现：输入组织、工具注册、sys_prompt、hooks  
  - 能解释 QwenPawAgent 初始化时各组件的责任边界  
  - 入口：[Module_03_Lesson_01_agent_core_loop.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_03_Lesson_01_agent_core_loop.md)

- Lesson 02：上下文工程 + 记忆管理 + 状态管理  
  - 理解“上下文窗口”和“长期记忆”在工程上的分工：压缩/裁剪 vs 检索/沉淀  
  - 能描述并实现一个最小可用的上下文更新策略与记忆检索策略  
  - 能在系统里识别关键状态（会话/任务/计划/工具审批）并建模为可观测状态机  
  - 入口：[Module_03_Lesson_02_context_memory_state.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_03_Lesson_02_context_memory_state.md)

### Module 04：工具、Skills、MCP 与多智能体协作（扩展与编排）

- Lesson 01：工具管理 + Skills + MCP + 多智能体协作  
  - 能设计工具接口、调用流程与选择策略（含失败模式）  
  - 能理解 Skills 的“封装形态”和“运行时加载方式”  
  - 能解释 MCP 客户端的生命周期管理与热更新策略  
  - 能给出一套可落地的多智能体协作模式（主从/团队/路由/仲裁）  
  - 入口：[Module_04_Lesson_01_tools_skills_mcp_collab.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_04_Lesson_01_tools_skills_mcp_collab.md)
- Lesson 02：多智能体协作的工程落地（机制 + 模式 + 反模式）  
  - 能区分“运行时多智能体（workspace 隔离）”与“协作多智能体（agent-to-agent 通信）”  
  - 能走通 inter-agent chat：list_agents/chat_with_agent/submit_to_agent/check_agent_task  
  - 能掌握 session_id/root_session_id 在跨 agent 协作里的作用与风险点  
  - 入口：[Module_04_Lesson_02_multi_agent_collaboration_patterns.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_04_Lesson_02_multi_agent_collaboration_patterns.md)

### Module 05：安全与治理（让 Agent 可在真实环境运行）

- Lesson 01：安全与治理（数据安全 / 权限控制 / 工具守卫 / 技能扫描）  
  - 理解“事前守卫 + 事中审批 + 事后审计”的分层治理方法  
  - 能把风险识别与审批结果回灌到对话上下文中，让 LLM 自己学会规避  
  - 入口：[Module_05_Lesson_01_security_and_governance.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_05_Lesson_01_security_and_governance.md)

### Module 06：API 路由层（系统边界与接口契约）

- Lesson 01：API 路由层设计  
  - 能组织路由模块、划分资源边界与版本/多租户策略  
  - 能理解 QwenPaw 的 agent-scoped 路由与上下文注入机制  
  - 能为 “agent + workspace + skills/tools/mcp” 设计稳定 API 契约  
  - 入口：[Module_06_Lesson_01_api_routing_layer.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_06_Lesson_01_api_routing_layer.md)

### Module 07：综合实践与总结（回到“总”）

- Lesson 01：典型案例实战 + 维度协同 + 评估与进阶  
  - 完成一个端到端的案例：从需求 → 架构 → 扩展点（Skills/Tools/MCP）→ 治理 → 验收  
  - 学会系统优化方法（性能、可靠性、可扩展性、安全）  
  - 给出学习成果评估与进阶路线（读源码/改需求/写测试/做发布）  
  - 入口：[Module_07_Lesson_01_capstone_and_summary.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_07_Lesson_01_capstone_and_summary.md)

### Module 08：Channels（渠道）与异步链路（把 Agent 接入真实世界）

- Lesson 01：Channels 的异步心智模型（读懂线程、事件循环与队列）  
  - 能解释 event loop / coroutine / task / async generator 在渠道链路中的角色  
  - 能理解 weixin 为何采用“后台线程 + 自建 event loop”，以及 thread-safe 入队的必要性  
  - 入口：[Module_08_Lesson_01_async_mentality_for_channels.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_08_Lesson_01_async_mentality_for_channels.md)
- Lesson 02：Weixin 一条消息的端到端链路（入参/出参/调用方逐节点串联）  
  - 能把平台消息 → native payload → AgentRequest → runner event → 回包的主链路串起来  
  - 能对每个关键节点说清调用方、入参形状、出参/副作用（以 weixin 为主例）  
  - 入口：[Module_08_Lesson_02_weixin_inbound_to_reply_end_to_end.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_08_Lesson_02_weixin_inbound_to_reply_end_to_end.md)
- Lesson 03：Console 对比 + 新渠道设计清单 + 如何验证  
  - 能对比 Console/Weixin 的模型差异，理解 BaseChannel/ChannelManager 的抽象边界  
  - 能按“必做/选做”清单设计并接入一个新渠道，并知道如何用现有测试/契约做验证  
  - 入口：[Module_08_Lesson_03_console_contrast_extension_and_tests.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_08_Lesson_03_console_contrast_extension_and_tests.md)
