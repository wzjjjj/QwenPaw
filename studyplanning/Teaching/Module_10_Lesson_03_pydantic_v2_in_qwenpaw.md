# Module 10 / Lesson 03：Pydantic v2：配置模型与 Schema 是怎么“在工程里跑起来”的

## 本课总览（Agent 框架为什么离不开 Pydantic）

在大模型应用工程里，“结构化”几乎就是稳定性的同义词：

- 配置要能校验：否则线上就是“字符串拼出来的炸弹”
- 接口要能约束：否则前后端/工具/模型之间会不断出现隐式不兼容
- 输出要可控：否则调试与治理无从谈起

QwenPaw 使用 Pydantic v2 把这些东西落在工程里：配置模型、路由 schema、provider 信息、plan 结构等。本课的目标是让你具备“能读懂、能手写、能扩展”的能力。

## 学习目标

- 能读懂并手写 Pydantic v2 的关键用法：`Field`、`default_factory`、`model_config`、`field_validator`、`model_validator`
- 能解释“配置加载/校验/默认值/兼容历史字段”的设计动机
- 能为一个真实配置/接口字段补齐校验，并能定位校验失败原因
- 能给 schema 设计“稳定边界”：哪些字段该 strict、哪些该 allow/ignore

## 先修知识

- 已掌握 Lesson 02 的 typing 基础（尤其是 `Literal`、`| None`）

## 本节要回答的关键问题

1. `Field(default_factory=...)` 为什么比 `[]/{}` 这种默认值写法更安全？
2. `model_config = ConfigDict(extra="ignore"|"allow")` 在工程里意味着什么边界策略？
3. `field_validator` 与 `model_validator` 的适用场景分别是什么？
4. 什么时候应该用 Pydantic，什么时候用 `TypedDict` 就够了？

## 核心概念（按“工程落地”组织）

### 1) BaseModel = “可运行的契约”

你可以把 `BaseModel` 当成三件事的合体：

- 数据结构定义（字段 + 类型 + 默认值）
- 运行时校验（输入是否合法）
- 文档与交互协议（Field 描述、枚举约束、前端/控制台可读）

QwenPaw 的 config 模型就是最典型的“工程级 BaseModel”：

- [ACPConfig._merge_default_agents](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/config.py#L104-L117)（`model_validator(mode="after")` 合并默认配置）
- [MatrixConfig.strip_trailing_slash](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/config.py#L318-L327)（`field_validator` 做字段级清洗）

### 2) 默认值：用 `default_factory` 避免共享可变对象

在工程里，最常见的隐蔽 bug 是“所有实例共享同一个 list/dict 默认值”。QwenPaw 在大量 schema 里都用 `default_factory` 避免这个坑：

- [ProviderInfo.models / extra_models](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/providers/provider.py#L93-L101)
- [PlanStateResponse.subtasks](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/plan/schemas.py#L22-L33)
- [SkillInfo.references / scripts](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/skills_manager.py#L89-L106)

### 3) `model_config`：你必须显式选择“边界严格度”

Pydantic v2 的 `model_config` 直接表达“外界输入”对你的系统能产生多大影响：

- `extra="ignore"`：忽略未知字段，兼容旧配置/前端新字段（更稳，但可能吞错）
- `extra="allow"`：接受未知字段并保留（更灵活，但更难治理）
- 其它策略：按模块需要取舍

在 provider 体系里可以看到更偏工程的配置：

- [ProviderInfo.model_config](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/providers/provider.py#L75-L84)（允许 arbitrary types，且关闭 validate_default，服务于测试/重载场景）

### 4) Schema 层 vs Config 层：两种“输入来源”，两种治理策略

建议你把 Pydantic 的使用按“输入来源”分两类：

- Config：来自磁盘/环境变量/用户配置，强调兼容与可迁移（需要合并默认、容错、迁移）
- Schema：来自 API/内部调用，强调契约稳定（更适合严格校验、明确错误语义）

QwenPaw 里两个代表性入口：

- Config： [config.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/config.py)
- API Schema： [plan/schemas.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/plan/schemas.py)

## 代码走读路线（建议按这个顺序）

- 从配置模型开始：看“默认合并 + 字段清洗”的两种 validator  
  - [config.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/config.py#L70-L117)  
  - [config.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/config.py#L318-L327)
- 再看 provider：理解 `model_config` 为什么要为测试/重载让路  
  - [provider.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/providers/provider.py#L75-L145)
- 再看 API schema：理解“响应结构化”如何让前端和调试变简单  
  - [plan/schemas.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/plan/schemas.py#L1-L66)
- 最后看 skills：理解返回给控制台/接口的“可读信息结构”  
  - [SkillInfo](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/skills_manager.py#L89-L106)

## 动手练习

### 练习 1（基础）：手写一个“最小配置模型”并加 1 个 validator

- 仿照 [MatrixConfig.strip_trailing_slash](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/config.py#L318-L327)
- 写一个字段清洗：去掉空格、规范化 URL、把空字符串转成 `None` 等

### 练习 2（进阶）：新增一个配置字段并贯通一条最小链路

- 选择一个你能解释清楚的配置点（例如某个 channel 的 timeout、某个 provider 的开关）
- 在 config BaseModel 中新增字段 + 校验
- 让它能从配置文件/接口进入系统，并在运行路径里被读取（加 1 行日志或可观察行为即可）

### 练习 3（挑战）：给错误语义“立规矩”

- 为一个可能失败的配置输入定义错误语义（ValueError / 自定义异常 / 错误码）
- 目标：让调用方（CLI/控制台/API）能区分“用户输入错误”与“系统错误”

## 验收清单

- [ ] 我能解释 `Field(default_factory=...)` 的必要性，并能指出它在项目里的 3 个真实用例
- [ ] 我能区分 field_validator 与 model_validator，并能在 config 里找到对应实例
- [ ] 我能解释 `extra="ignore"/"allow"` 的边界含义，并能说出它对治理的影响
- [ ] 我能新增一个字段（带校验）并贯通最小链路，且能用测试或行为验证它生效

## 下一课预告

下一课进入“装饰器与上下文管理器”：你会从 click 命令栈、Pydantic validator、lru_cache、以及资源释放模式（contextmanager/suppress）理解框架式代码的隐式控制流，做到“读得懂、改得稳”。
