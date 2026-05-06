# Module 06 / Lesson 01：API 路由层的设计（总分总）

## 本课总览（总）

对后端开发者来说，Agent 项目能不能持续迭代，关键在于有没有稳定的系统边界：  
**API 路由层**就是这条边界，它决定了：

- 哪些能力对外暴露（chats/skills/tools/mcp/config/plan）
- 如何做多租户/多 agent 隔离（agent-scoped）
- 如何保证接口契约稳定（请求/响应 schema、错误语义、版本演进）

这一课通过 QwenPaw 的 FastAPI 路由组织方式，讲清 API 路由层的模块化设计、agent 上下文注入、以及你在自己的 Agent 项目里应如何设计 API 契约。

## 学习目标

- 能组织一个 Agent 服务的 API 路由模块：按资源边界拆分、统一聚合、支持扩展
- 能理解 QwenPaw 的 agent-scoped 路由机制，以及它如何与 multi-agent runtime 协同
- 能为技能/工具/MCP/对话等核心资源设计稳定 API 契约与错误语义

## 先修知识

- 已理解系统分层（路由层 / runner / agent / 能力层 / 治理层）
- 有 FastAPI 的基础概念

## 本节要回答的关键问题

1. 为什么 Agent 项目的 API 更需要“契约稳定”？哪些字段最容易变？
2. 如何设计 agent-scoped 的路由，以支持一个进程服务多个 agent？
3. API 层应该承担哪些责任，哪些必须下沉到 runner/agent？
4. 面对异步任务、审批流、热更新，API 应该怎么表达状态？

## 核心概念（分）

### 1) 路由组织：聚合点与资源边界

QwenPaw 把路由模块按资源边界拆分（agents/config/skills/tools/mcp/...），再在聚合入口统一 include：  
- [routers/__init__.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/__init__.py#L4-L53)

这种组织方式的工程收益：

- 新增模块不影响已有模块（低耦合）
- 聚合点清晰（便于文档、权限与中间件统一处理）
- 资源边界明确（例如 skills 与 tools 的 API 语义不同）

### 2) agent-scoped：多 agent 隔离的路由策略

多 agent 隔离常见有两种方式：

- **路径隔离**：`/agents/{agentId}/...`
- **Header 隔离**：`X-Agent-Id: ...`

QwenPaw 采用两者兼容，并用中间件把 agent_id 注入上下文变量，供 runner/manager 使用：  
- [AgentContextMiddleware.dispatch](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L18-L63)

然后把一整套 router 挂载到 `/agents/{agentId}` 下：  
- [create_agent_scoped_router](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L66-L104)

这让“同一套 API”在多 agent 下复用，同时又能做到隔离。

### 3) 路由层与 runner/agent 的职责分离（非常关键）

建议你用下面规则强制分层：

- API 层：只处理 HTTP 语义（鉴权、参数校验、返回结构、错误码、流式传输协议）
- Runner 层：处理一次请求生命周期（命令短路、skills 注入、流式队列、异常兜底）
- Agent 层：处理推理与工具调用（含上下文/记忆/守卫）

QwenPaw 的 DynamicMultiAgentRunner 体现了这种分离：API 只负责进来，真正的“选择哪个 workspace/runner”由动态 runner 完成：  
- [_app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L126)

### 4) API 契约：请求/响应、错误语义、状态表达

Agent 项目的 API 契约建议具备三类统一语义：

- **成功响应**：固定 envelope（例如 data + metadata），避免前端/客户端猜字段  
- **错误响应**：结构化错误类型（type）、可定位 detail、可恢复建议（hint）  
- **状态响应**：长任务/审批流需要明确状态机字段（pending/running/completed/failed）

你可以参考 QwenPaw 在 skills 扫描失败时的结构化错误返回，它不仅有 HTTP status，还能携带 findings 细节：  
- [_scan_error_payload](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/skills.py#L70-L99)

### 1.1.5 补充：ACP 配置接口、降级继承与时区规范化（属于“契约稳定”的一部分）

一些看起来“偏配置/偏控制台”的需求，本质也是 API 契约稳定性问题：前端/UI 只是表现层，真正需要稳定的是后端的配置接口与默认行为。

- **ACP Agent 重命名/删除（控制台能力的后端落点）**：ACP agents 是一个 dict 配置集合；控制台做“重命名/删除”时，本质是在更新这份配置并触发热重载  
  - 获取/更新 ACP config：`GET/PUT /api/config/acp`  
  - 获取/更新单个 ACP agent：`GET/PUT /api/config/acp/{agent_name}`  
  - [config.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/config.py#L394-L505)
- **ACP 降级继承**：当 workspace 缺少 `agent.json` 需要生成降级配置时，会从根配置继承 ACP 配置，避免“降级后 ACP 失效”  
  - [build_fallback_agent_profile_config](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/config.py#L1625-L1677)
- **时区名称规范化**：对非标准/旧别名时区做映射，最终落到 IANA 标识符（例如 `Asia/Beijing` → `Asia/Shanghai`），让定时任务、日志时间与跨环境一致  
  - [timezone.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/timezone.py#L19-L108)

## 代码走读路线（分）

1. 路由聚合入口：  
   - [routers/__init__.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/__init__.py#L4-L53)
2. agent 上下文注入与 agent-scoped 挂载：  
   - [agent_scoped.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/agent_scoped.py#L15-L104)
3. 动态 runner 如何按 agent_id 选择 workspace：  
   - [_app.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/_app.py#L71-L126)

## 动手练习（可执行）

1. 为你自己的项目设计一组 API 路由（只写路径与方法即可）：chats、skills、tools、mcp、config、plan。
2. 为其中 2 个接口写出请求/响应 schema（用 pydantic 或 OpenAPI 描述均可）。
3. 设计一个“长任务状态接口”：定义 task_id、status、progress、error、result 的字段。
4. 设计一个“审批流接口”：定义 approval_id、tool_name、status、findings、decision_reason。

## 验收清单

- [ ] 能解释 API 层与 runner/agent 的职责边界，并能指出 QwenPaw 的对应实现点
- [ ] 能描述 agent-scoped 机制：从 path/header 取 agent_id → 注入上下文 → 复用 router
- [ ] 能写出一份稳定的 API 契约草案（含错误与状态表达）

## 本课小结（总）

路由层是系统边界，边界稳定，系统才可演进。  
当你用 agent-scoped 把多 agent 隔离做成“路径/上下文注入”，再让 runner/agent 承担真正的执行语义，你就能在不破坏接口的情况下持续扩展能力与治理策略。  
下一课进入综合实践：把所有维度协同起来，完成一个典型 Agent 项目案例演练，并给出优化与进阶建议。

## 下一课预告

- 进入 [Module_07_Lesson_01_capstone_and_summary.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_07_Lesson_01_capstone_and_summary.md)
