# Module 01 / Lesson 03：配置与会话持久化（Config & Sessions）

## 学习目标

- 能说清 QwenPaw 的配置体系：配置文件在哪里、如何加载/校验/降级、为什么要做 mtime 缓存与自动修复
- 能读懂 QwenPaw 的会话持久化语义：session 文件名跨平台兼容、JSON 损坏恢复、以及“最坏情况下系统如何继续跑”
- 能写出两张“失败语义表”：配置失败语义与会话失败语义（这是你后续做可靠性/治理的基础）

## 先修知识

- Pydantic v2：BaseModel、Field、validator 基本用法
- 文件 I/O 基础：JSON、编码、Windows 文件名限制
- async/await：理解为什么“磁盘 I/O 阻塞 event loop”是问题（不需要会优化）

## 本节要回答的关键问题

1. 配置为什么要“允许修复/移除坏字段/回退默认值”？这会带来什么 trade-off？
2. `load_config` 的 mtime 缓存解决什么问题？它会带来什么一致性风险？
3. 会话文件为什么容易坏（尤其在并发/异常退出场景）？坏了之后系统应该怎么表现才算“工程上可接受”？
4. Windows 文件名限制为什么会影响 session_id/user_id 的设计？在哪里做了兜底？

## 核心概念：把“配置/会话”当作系统的稳定地基

在一个会长期运行的 Agent 系统里：
- **配置**决定“系统应该怎么运行”（开关、阈值、provider、工具白名单、安全级别）
- **会话**决定“系统记得什么”（状态、历史、任务、审批关联）

这两者有一个共同点：它们都必须具备明确的失败语义（否则线上排障会变成玄学）。

## 代码走读路线

### 1) 配置读取与缓存：`load_config` 的 mtime 缓存与校验降级

- 配置加载：[`load_config`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/utils.py#L538-L576)
- 校验与错误处理：[`_load_and_validate_config`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/utils.py#L502-L536)

你需要抓住 3 个工程点：
- **mtime 缓存**：文件没变就不重复读盘（降低 I/O，提升接口响应速度）
- **auto-repair + bad field removal**：JSON 语法损坏会尝试修复；校验失败会尝试删除坏字段再二次校验
- **最后兜底**：如果都不行，回退到默认 `Config()`（系统可运行优先）

### 2) 配置模型：Pydantic v2 负责“结构 + 约束”

配置模型定义在：
- [config/config.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/config.py)

本课不要求你把全部字段背下来，只要求你建立两层认知：
- **结构层**：哪些字段属于“核心运行时”/“channels”/“providers”/“agents”
- **约束层**：哪些字段有校验器/默认值/兼容逻辑（比如 agent_id 的合法字符集）

一个典型例子是 agent_id 约束（会影响 workspace 路由、目录命名、日志过滤）：
- 模式与长度/保留字校验：[`validate_agent_id`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/config.py#L150-L189)

### 3) 会话持久化：SafeJSONSession 的跨平台文件名与 JSON 损坏恢复

- 会话实现：[`SafeJSONSession`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/session.py#L78-L176)

你需要抓住 2 个“很工程”的坑：
- **Windows 文件名非法字符**：`:` `*` `?` 等会直接导致写文件失败，因此必须 sanitize
  - 规则：[`sanitize_filename`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/session.py#L63-L76)
  - 路径拼接：[`_get_save_path`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/session.py#L97-L110)
- **JSON 损坏恢复**：并发写/异常退出常见症状是“尾部多了垃圾字符”
  - 恢复策略：[`_safe_json_loads`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/session.py#L24-L60)

## 失败语义表（你要写出来的两张表）

### 表 1：配置失败语义（Config Failure Semantics）

| 失败类型 | 典型原因 | 系统行为（降级策略） | 风险/代价 |
| --- | --- | --- | --- |
| 配置文件不存在 | 首次启动/路径错误 | 返回默认 `Config()` | 默认值可能不符合预期（但能跑） |
| JSON 语法错误 | 手改文件、写入中断 | 尝试 repair；失败则备份并回退默认 | 修复可能误判；备份后丢用户配置 |
| 校验失败（字段类型/范围错） | 版本升级字段变化、手改错 | 尝试移除坏字段后再校验；不行则备份并回退默认 | 可能“悄悄忽略”部分配置 |

对照代码证据：
- 二次校验与移除坏字段：[`_load_and_validate_config`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/utils.py#L502-L536)
- 最终兜底默认：[`load_config`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/utils.py#L549-L576)

### 表 2：会话失败语义（Session Failure Semantics）

| 失败类型 | 典型原因 | 系统行为（恢复策略） | 风险/代价 |
| --- | --- | --- | --- |
| 文件名非法（Windows） | session_id/user_id 带 `:` 等 | sanitize 替换为 `--` | 路径可预测性下降；需确保唯一性 |
| JSON 尾部垃圾 | 并发写、进程中断 | raw_decode 截取首个合法对象 | 可能丢失最后一次写入的增量 |
| JSON 完全损坏 | 文件被截断/覆盖 | 返回 `{}` 并记录 warning | “看起来正常但历史丢了” |

对照代码证据：
- sanitize：[`sanitize_filename`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/session.py#L63-L76)
- 损坏恢复：[`_safe_json_loads`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/session.py#L24-L60)

## 动手练习

### 练习 1（基础）：写出你自己的两张失败语义表

要求：
- 配置失败语义表至少覆盖：缺文件、坏 JSON、校验失败
- 会话失败语义表至少覆盖：Windows 文件名非法、尾部垃圾、完全损坏

验收：
- 每个“系统行为”都必须给出 1 个代码锚点作为证据（路径 + 行号）

### 练习 2（进阶）：设计“最小不变量清单”

写出一份你认为必须长期成立的“不变量”清单（至少 6 条），例如：
- `agent_id` 一旦绑定到请求，就不能在 runner 中途被覆盖
- `session_id` 与 `user_id` 不应包含平台非法字符（或必须经过 sanitize）
- 配置校验失败不得让服务整体不可用（必须能回退默认启动）

验收：
- 每条不变量都写清：破坏它会出现什么现象（可观察的）

## 验收清单

- [ ] 能解释 `load_config` 为什么要做 mtime 缓存、为什么要允许降级
- [ ] 能说清“配置失败”三类场景下系统如何表现，并指出源码证据
- [ ] 能解释 SafeJSONSession 的两个坑（Windows 文件名 + JSON 损坏），并指出源码证据
- [ ] 已完成两张失败语义表 + 最小不变量清单

## 下一课预告

下一模块会把“配置/会话”这套稳定地基接到“队列/turn 语义”上：为什么需要 TaskTracker/ChannelManager，如何定义“一个 turn 的状态机”，以及流式 chunk/事件是怎么变成可重连、可停止、可审计的产品语义。

入口文件（提前标记）：
- [`TaskTracker`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/task_tracker.py#L34-L220)
- [`ChannelManager`](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L68-L110)

