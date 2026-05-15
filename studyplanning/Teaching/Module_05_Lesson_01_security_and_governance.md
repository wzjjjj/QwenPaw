# Module 05 / Lesson 01：安全与治理（总分总）

## 本课总览（总）

Agent 项目一旦进入“能执行工具”的阶段，安全问题会从“内容安全”升级为“系统安全”：

- 读写文件、执行 shell、访问网络、调用外部服务都可能造成数据泄露或资源破坏
- LLM 可能被提示词注入诱导做越权操作
- 多智能体协作会放大攻击面与误操作传播

这一课给出一套可落地的治理框架：**事前守卫（Guard）→ 事中审批（Approval）→ 事后审计（Audit）**，并用 QwenPaw 的 ToolGuardEngine/ToolGuardMixin 与技能安全扫描作为具体落点。

## 学习目标

- 能识别 Agent 项目中的关键风险面，并把风险映射到治理点（入口/工具/技能/存储/API）
- 能理解并复用 QwenPaw 的工具守卫设计：guard scope、deny list、审批级别、并发安全
- 能设计技能安装/导入的安全扫描流程与结构化错误返回

## 先修知识

- 已理解工具/skills/MCP 的扩展点
- 能读懂“守卫引擎 + mixin 拦截 + API 层返回”的组合模式

## 本节要回答的关键问题

1. 为什么“把规则写进 prompt”不够？治理必须落在哪些系统边界？
2. 工具守卫应该检查什么：参数、目标路径、命令模式、网络目的地、资源预算？
3. 审批流如何设计才能既安全又不毁体验？如何做到可解释？
4. 技能系统为什么也需要安全扫描？扫描失败如何回传给用户与模型？

## 核心概念（分）

### 1) 风险建模：从攻击面到治理点

建议你把风险分成四类（每类都要有治理点）：

- **数据安全**：敏感文件、密钥、隐私数据、日志泄露
- **权限越界**：工具调用越权、跨 workspace 访问、跨 agent 操作
- **资源滥用**：fork bomb、无限循环、长时间网络/CPU 占用
- **提示词注入**：工具调用参数被外部内容诱导（网页、文档、工具输出）

对应治理点一般包括：

- API 路由层：鉴权、租户隔离、agent_id 绑定、审计
- 工具执行层：参数守卫 + deny/guard scope + 审批流
- Skills 安装层：静态扫描 + 依赖/脚本风险检查
- 存储层：密钥隔离、最小权限、加密（如有）

### 2) 工具守卫：事前风险识别（Guard Engine）

QwenPaw 的守卫引擎把“规则检查”抽象为 guardians，并汇总 findings：  
- [engine.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/security/tool_guard/engine.py#L54-L235)

关键设计点：

- 可配置开关：env > config > default（默认开启）  
  - [_guard_enabled](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/security/tool_guard/engine.py#L36-L52)
- guarded scope / denied tools：同一套引擎可以对不同工具做不同策略  
  - [engine.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/security/tool_guard/engine.py#L139-L172)
- guardian 可扩展：可注册自定义 guardian（例如你自己的业务规则）  
  - [register_guardian](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/security/tool_guard/engine.py#L116-L119)

### 3) 审批流：事中决策（ToolGuardMixin）

把审批放在工具执行前的统一入口，能保证覆盖面与一致性。QwenPaw 用 Mixin 覆盖 `_acting` 实现这一点：  
- [tool_guard_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L138-L176)

这里还有一个重要的工程细节：并发安全。  
Mixin 里用 `_tool_guard_lock` 串行化“守卫决策块”，避免 `parallel_tool_calls=True` 时共享属性竞态，但把实际执行放在锁外以保证吞吐：  
- [tool_guard_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L149-L176)

你在自己的项目里可以直接复用这种思路：**锁住决策，不锁执行**。

### 4) 技能安全扫描：把风险前移到“安装时”

技能风险比工具更隐蔽，因为它可能携带：

- 引导模型越权的提示词
- 潜在恶意脚本或依赖
- 硬编码密钥、数据外泄逻辑

QwenPaw 在 skills_manager 引入扫描：  
- [skills_manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/skills_manager.py#L27-L30)

并在 API 层把扫描失败变成结构化 422 响应（含 findings，便于 UI 展示与排障）：  
- [_scan_error_payload](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/skills.py#L70-L99)

这体现了治理的另一个核心原则：**失败也要可结构化传递**，否则模型无法自我修正，用户也无法定位原因。

## 代码走读路线（分）

1. ToolGuardEngine：默认 guardians、guard scope、deny list  
   - [engine.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/security/tool_guard/engine.py#L85-L172)
2. ToolGuardMixin：_acting 的拦截与并发锁策略  
   - [tool_guard_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L138-L176)
3. Skills API：扫描失败如何返回结构化 findings  
   - [skills.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/routers/skills.py#L70-L111)

## 动手练习（可执行）

1. 列出你项目中最危险的 5 个工具（或能力），为每个写出：风险、守卫规则、是否需要审批、审计字段。
2. 设计一个“deny list”与“guard scope”的策略：哪些工具永远禁止，哪些工具必须审批，哪些工具只做 always_run 检查。
3. 设计技能扫描的最小规则集：至少包含“硬编码密钥检测、危险命令检测、提示词注入模式检测”三类。

### 进阶练习（提升）

1. 以 [tool_guard_mixin.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L138-L176) 的“锁住决策、不锁执行”为参照，写出你自己的并发安全方案：哪些共享状态必须串行化，哪些操作必须放锁外避免拖垮吞吐。
2. 设计一份“审批可解释性模板”：审批提示里必须包含哪些字段（tool、参数摘要、风险点、建议替代方案、回滚说明），并说明这些字段应该来自 guard engine 的哪一层输出。

## 验收清单

- [ ] 能给出一张风险地图：风险分类 → 治理点 → 责任模块
- [ ] 能解释 QwenPaw 工具守卫的三件事：guard engine、mixin 拦截、审批并发安全
- [ ] 能说明技能扫描失败为什么要返回结构化 findings

## 本课小结（总）

治理不是“加个开关”，而是把风险从“事后救火”前移到“事前决策”，并让系统在失败时仍然可解释、可审计、可恢复。  
下一课我们进入 API 路由层设计：系统边界与接口契约决定了你能否稳定迭代、能否支持多 agent 与多端接入。

## 下一课预告

- 进入 [Module_06_Lesson_01_api_routing_layer.md](file:///d:/编程学习记录/QwenPaw/studyplanning/Teaching/Module_06_Lesson_01_api_routing_layer.md)
