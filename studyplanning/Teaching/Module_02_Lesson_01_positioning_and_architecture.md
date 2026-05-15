# Module 02 / Lesson 01：项目定位与整体架构（总分总）

## 本课总览（总）

在 Agent 项目里，**定位**不是市场文案，它会直接决定：

- 你需要哪些能力（工具/技能/多渠道/多智能体）
- 你需要哪些治理（安全、权限、审计）
- 你系统的边界在哪里（本地/云、单用户/多用户、多 workspace/多 agent）

这一课把“项目定位”落到“系统架构”。我们用 QwenPaw 作为样例，建立一个对后端开发者友好的分层模型，并指向真实代码入口。

## 学习目标

- 能完成一次 Agent 项目的场景分析：任务类型、数据边界、风险面、用户交互形态
- 能写出价值主张与目标用户群（并能反推系统能力与约束）
- 能给出系统分层架构，并在 QwenPaw 中找到模块划分与交互机制

## 先修知识

- 已完成 Module 01，具备“入口 → runner → agent → 扩展/治理”的全局坐标

## 本节要回答的关键问题

1. 为什么很多 Agent 项目“越做越乱”？定位与架构的关系是什么？
2. 一个工程化 Agent 系统应该按什么维度分层？每层对上/对下的契约是什么？
3. QwenPaw 如何处理多智能体：路由/隔离/懒加载/热重载分别放在哪？
4. 模块间交互更适合“函数调用”还是“事件驱动/消息传递”？如何混用？

## 核心概念（分）

### 1) 项目定位：应用场景、价值主张、目标用户（工程化写法）

用 3 个问题写定位（建议你写在自己的项目 README 里）：

1. **用户要完成什么任务闭环？**（输入→处理→输出→反馈→持续）  
2. **系统的可信边界在哪？**（数据是否出本地；工具是否能执行系统命令；是否需要审批）  
3. **目标用户的“默认环境”是什么？**（后端开发者/普通用户；本地/云；是否多端接入）

以 QwenPaw 为例，它的定位决定了几个关键工程选择：

- **多渠道接入 + 本地与云都可部署**：意味着 API 层要稳定、配置要可迁移、运行时要可热更新
- **Skills 扩展**：意味着能力要“可封装/可安装/可扫描/可启停”
- **多层安全防护**：意味着必须在工具调用前有可解释的守卫与审批流

### 2) 项目整体架构：推荐分层与职责边界

这里给出一个对后端/算法开发者更友好的分层方式（你可以直接复用）：

1. **API 路由层（System Boundary）**  
   - 对外暴露稳定接口：chats/skills/tools/mcp/config/plan  
   - 负责鉴权（可选）、多租户/多 agent 上下文注入  
   - QwenPaw 聚合路由入口：[routers/__init__.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/__init__.py#L4-L53)
2. **运行器层（Runner / Orchestrator）**  
   - 驱动一次请求的生命周期：命令识别、skills 短路、流式输出、错误兜底  
   - QwenPaw runner： [runner.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L107-L259)
3. **工作区/多智能体管理层（Workspace & Multi-Agent Runtime）**  
   - 解决“同一进程里跑多个 agent”的隔离与生命周期（start/stop/reload）  
   - QwenPaw 懒加载去重：[multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)
4. **Agent 层（Decision & Action Loop）**  
   - 组装模型、sys_prompt、工具、skills、context/memory、hooks、守卫  
   - QwenPawAgent 初始化：[react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L79-L215)
5. **能力层（Tools / Skills / MCP / Knowledge）**  
   - 工具：函数级能力（读写文件、shell、搜索、浏览器、委托其他 agent）  
   - Skills：面向任务的封装与分发（可安装、可配置、可扫描）  
   - MCP：外部工具/服务的协议化接入与生命周期管理  
   - QwenPaw skills API： [skills.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/skills.py#L64-L131)
6. **治理与基础设施层（Security / Storage / Observability）**  
   - 事前守卫：工具调用风险识别与审批  
   - 技能扫描：安装前安全扫描与结构化错误返回  
   - QwenPaw 工具守卫入口： [tool_guard_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L138-L176)

### 3) 模块间交互机制：三种常见模式（以及在 QwenPaw 的落点）

1. **同步调用（最直接）**：函数调用/方法调用  
   - 适合：同进程、强一致、短路径（如 agent 内部组装）  
   - 示例：QwenPawAgent 初始化时注册工具/skills：[react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L145-L183)
2. **基于 Hook 的事件点（可插拔）**：在关键生命周期插入自定义逻辑  
   - 适合：上下文压缩、工具结果裁剪、打印/追踪等横切逻辑  
   - 示例：context manager hooks 注册：[react_agent.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L429-L452)
3. **管理器模式（集中生命周期）**：把外部资源/多实例的生命周期集中到 manager  
   - 适合：多 agent、MCP 客户端、channels、providers 等  
   - 示例：MCP 热更新与替换：[manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/mcp/manager.py#L23-L132)

## 代码走读路线（分）

1. 看 API 路由如何聚合： [routers/__init__.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/__init__.py#L4-L53)  
2. 看 agent-scoped 路由如何把同一套 API “挂到不同 agent 下”： [agent_scoped.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L66-L104)  
3. 看多 agent 的 workspace 如何创建与缓存： [multi_agent_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/multi_agent_manager.py#L42-L137)  
4. 看 runner 如何短路 skills（这体现了“能力层”和“执行层”的边界）： [_maybe_inject_skill](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L185-L259)

## 动手练习（可执行）

目标：输出你自己的“定位 → 架构”一页纸。

1. 写一段定位（不超过 10 行）：场景、价值、用户、数据边界、风险面。
2. 画出 6 层架构图，并为每层写 1 条“只做什么/不做什么”的边界描述。
3. 选择一个扩展点（Tools/Skills/MCP/多 agent），写下它应该在哪一层实现，以及为什么。

### 进阶练习（提升）

1. 为你的架构补齐“失败模式表”：至少列 6 个失败（模型超时、工具超时、上下文爆炸、审批卡住、渠道乱序、配置热更新导致竞态），并写出落点（哪一层兜底、返回什么、怎么观测）。
2. 选一条链路做“依赖倒置”检查：列出你希望上层依赖下层的接口（协议/抽象），而不是依赖具体实现的地方（例如 runner 只依赖 agent 接口、channel 只依赖 process handler）。

## 验收清单

- [ ] 能在不提框架名的情况下描述系统分层（只用职责语言）
- [ ] 能在 QwenPaw 中指出：API 聚合点、agent-scoped 路由点、multi-agent 管理点
- [ ] 有一份你自己的“定位 → 架构”一页纸（可贴到项目 README）

## 本课小结（总）

定位决定边界，边界决定架构。  
当你用“层的职责”把系统切开，你就得到了一套可扩展的工程骨架：新增能力放在能力层；改变执行策略放在 runner/agent；跨租户/多 agent 的隔离放在 workspace 与路由；治理放在守卫与扫描。  
下一课我们进入 Agent 主体：把模型组装成一个“能稳定执行”的循环系统。

## 下一课预告

- 进入 [Module_03_Lesson_01_agent_core_loop.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_03_Lesson_01_agent_core_loop.md)
