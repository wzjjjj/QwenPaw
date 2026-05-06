# QwenPaw Agent 项目学习路线（总分总）

本课程以 **Agent 项目开发者**视角，使用“总分总”的结构化方式，系统学习一个可落地的 Agent 工程（以 QwenPaw 为样例）的核心维度：从项目定位 → 架构分层 → Agent 主循环 → 上下文/记忆 → 工具/Skills/MCP → 多智能体协作 → API 路由层 → 安全治理 → 综合实战与优化。

你会收获的不是“概念堆叠”，而是能直接迁移到自己项目的工程方法：如何划分模块、如何设计扩展点、如何把 LLM 的不确定性装进可控的系统边界里。

## 如何使用这套材料

- 推荐阅读顺序：先读 [Course\_Syllabus.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Course_Syllabus.md)，再按模块逐课推进。
- 每一课都包含：学习目标、关键问题、代码走读路线、关键代码讲解（带可点击锚点）、动手练习、验收清单、下一课预告。
- 默认受众：后端/算法开发者（FastAPI + 运行器 + 任务/工具/安全/配置视角）。前端控制台仅在“API 路由层”里做最低必要解释。

## 总体地图（从“总”到“分”）

为了让你在阅读细节前先有全局坐标，这里给出 QwenPaw 的“最短主链路”：

1. **HTTP 入口与多智能体路由**：FastAPI app 装配 + 中间件注入 agent\_id
   - [\_app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L216)（动态 runner）
   - [agent\_scoped.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L15-L63)（从 path/header 注入 agent\_id）
2. **Workspace/Runner**：按 agent\_id 懒加载工作区，Runner 驱动一次 query 的完整生命周期
   - [multi\_agent\_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L22-L137)
   - [runner.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L107-L259)（命令/skills 注入与短路）
3. **Agent 主体**：ReActAgent + 工具/skills/memory/context/guard 的组装
   - [react\_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L79-L215)（QwenPawAgent 初始化）
   - [react\_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L216-L410)（工具注册、skills 注册、sys\_prompt 构建）
4. **安全与治理**：工具调用前拦截（deny/guard/approval），把风险从“事后补救”前移到“事前决策”
   - [tool\_guard\_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L138-L280)（\_acting 拦截与决策）
   - [engine.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/security/tool_guard/engine.py#L54-L235)（guardians 编排与规则加载）

## 课程目录（跳转）

- Module 01：总体概述（发展现状 / 学习框架 / 学习路径）
  - [Module\_01\_Lesson\_01\_overview.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_01_Lesson_01_overview.md)
- Module 02：项目定位与整体架构（应用场景 / 分层设计 / 模块交互）
  - [Module\_02\_Lesson\_01\_positioning\_and\_architecture.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_02_Lesson_01_positioning_and_architecture.md)
- Module 03：Agent 主体与上下文/记忆/状态（决策系统 / 推理机制 / 上下文工程 / 记忆管理 / 状态管理）
  - [Module\_03\_Lesson\_01\_agent\_core\_loop.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_03_Lesson_01_agent_core_loop.md)
  - [Module\_03\_Lesson\_02\_context\_memory\_state.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_03_Lesson_02_context_memory_state.md)
- Module 04：工具、Skills、MCP 与多智能体协作（工具管理 / Skills 体系 / MCP 集成 / 多智能体协作）
  - [Module\_04\_Lesson\_01\_tools\_skills\_mcp\_collab.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_04_Lesson_01_tools_skills_mcp_collab.md)
  - [Module\_04\_Lesson\_02\_multi\_agent\_collaboration\_patterns.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_04_Lesson_02_multi_agent_collaboration_patterns.md)
- Module 05：安全与治理（数据安全 / 权限控制 / 风险识别与审批）
  - [Module\_05\_Lesson\_01\_security\_and\_governance.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_05_Lesson_01_security_and_governance.md)
- Module 06：API 路由层设计（路由组织 / agent-scoped / 接口契约）
  - [Module\_06\_Lesson\_01\_api\_routing\_layer.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_06_Lesson_01_api_routing_layer.md)
- Module 07：综合实践与总结（案例演练 / 协同优化 / 评估与进阶）
  - [Module\_07\_Lesson\_01\_capstone\_and\_summary.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_07_Lesson_01_capstone_and_summary.md)
- Module 08：Channels（渠道）与异步链路（收消息 / 串行消费 / 回包 / 扩展与验证）
  - [Module\_08\_Lesson\_01\_async\_mentality\_for\_channels.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_08_Lesson_01_async_mentality_for_channels.md)
  - [Module\_08\_Lesson\_02\_weixin\_inbound\_to\_reply\_end\_to\_end.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_08_Lesson_02_weixin_inbound_to_reply_end_to_end.md)
  - [Module\_08\_Lesson\_03\_console\_contrast\_extension\_and\_tests.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_08_Lesson_03_console_contrast_extension_and_tests.md)

## 学习建议（偏后端视角）

- 先把“边界”想清楚：Agent 不是模型，是一个包含 **输入/输出契约、状态机、工具/权限、存储、可观测性** 的系统。
- 读代码从“入口”开始：请求如何进来 → agent\_id 如何决定 workspace → runner 如何组装 agent → agent 如何调用工具/skills → 安全如何拦截。
- 练习优先做“可验证”的：改一个开关、观察一个行为变化、写一个最小技能、设计一个接口契约并能被 runner/agent 消费。

## 课程终点（回到“总”）

当你完成本课程，你应能：

- 画出自己的 Agent 项目架构图（至少包含：入口/路由、runner、agent、tools、skills、memory/context、security、observability）
- 为每个核心维度给出“工程落点”：对应模块、接口、数据结构、失败模式与测试策略
- 在不改动核心循环的情况下，扩展新工具、新 skills、新 MCP 客户端或新 agent
