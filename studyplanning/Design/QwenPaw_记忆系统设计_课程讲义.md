# QwenPaw 记忆系统设计：从三大主流框架的分析方法到可落地实现（课程讲义）

本讲义借鉴一篇关于 OpenClaw / Claude Code / Hermes Agent 的对比分析方法：用同一套问题框架拆记忆系统，再把这套框架套到 QwenPaw 上，最后落到“你如何自己设计一套记忆系统”。  
参考文章：AI Agent 架构设计（一）：记忆系统设计（OpenClaw、Claude Code、Hermes Agent 对比）

---

## 学习目标

- 能用“四个架构问题”评估任意 Agent 记忆系统：存什么 / 存在哪 / 怎么取 / 怎么管
- 能用“四层记忆模型”划分实现：in-context / external / episodic / parametric
- 能把 QwenPaw 的记忆系统画成一条链路：注入点、存储点、检索点、压缩点、回放点
- 能为自己的项目写出一套可执行的记忆策略：写入策略、检索策略、更新/衰减策略、失败兜底策略

## 先修知识

- 了解 Agent 主循环（Reasoning/Acting/Replying 的阶段概念）
- 能读懂“hook 注入”与“可插拔后端接口”

---

## 一、分析框架：四个架构问题 + 四层记忆模型

### 1) 四个架构问题（工程视角）

1. 存什么：哪些信息值得长期保留？哪些应该只留在短期上下文？
2. 存在哪：用什么介质与格式？生命周期是什么？
3. 怎么取：什么时候取？用什么算法取？召回多少？如何去噪？
4. 怎么管：如何压缩、更新、淘汰？如何保证“失败不致命”？

### 2) 四层记忆模型（能力视角）

- **上下文记忆（In-context）**：当前模型调用可见的 token 窗口内容
- **外部记忆（External）**：模型外持久化存储（文件/DB/向量库），按需检索注入
- **情景记忆（Episodic）**：带结构的“发生过什么、怎么做的、结果如何”（用于复盘与迁移学习）
- **参数记忆（Parametric）**：模型权重内知识（不可运行时更新）

这个讲义的核心做法：先用“四问”拆系统，再用“四层”对齐实现落点，避免把“聊天历史”“长期偏好”“任务经验”混成一锅粥。

---

## 二、三大主流框架的“设计哲学”要点（用来对标 QwenPaw）

这一段不是复述框架细节，而是抽出可迁移的设计决策。

### 1) OpenClaw：文件系统即记忆（透明、可控）

- **唯一真理**：没写进文件，就不算记住
- **两层文件结构**：短期日志（追加） + 长期精华（MEMORY.md）
- **混合检索**：语义 + 关键词并行，互补精确与泛化
- **Compaction 的最大风险**：对话中约定没落盘，压缩后就丢；用 “Memory Flush” 在压缩前先落盘缓解

### 2) Claude Code：上下文工程优先（预算调度）

- 把“记忆系统”更多当作 **token 预算分配与分层注入机制**
- 固定层走缓存、条件层按需注入；用目录层级编码规则作用域
- 让 Agent 感知自身 token 状态，主动调整策略（例如压缩前完成关键步骤）

### 3) Hermes：四层分离，情景记忆是核心（长期演化）

- 热记忆（小而精）常驻注入；历史归档按需检索
- 情景记忆以“可复用技能/流程”为核心，渐进加载（先索引、再全文）
- 经验会自我更新：使用中迭代 Skill 文档

对你最有用的不是“抄实现”，而是：识别你要做的产品类型，然后选哲学。

---

## 三、QwenPaw 的记忆系统全景图（先画地图）

QwenPaw 的记忆系统是一个“混合体”：

- 像 OpenClaw：长期记忆落在文件（MEMORY.md、memory/*.md），并提供语义检索工具
- 像 Claude Code：强上下文治理（压缩、工具结果裁剪、注入策略），并且把治理挂在生命周期 hook 上
- 像 Hermes：开始具备“分层”的雏形（会话状态、对话归档、长期精华、检索注入），但情景记忆（可复用 workflow/skills 的自动沉淀）仍处于相对早期

接下来按“四层记忆”逐层拆解。

---

## 四、按“四层记忆模型”拆 QwenPaw

### 第 1 层：In-context（当前上下文窗口）

#### 4.1 载体：AgentContext（内存消息容器 + 压缩摘要）

- AgentContext 继承 InMemoryMemory，并维护 `content`（消息列表）与 `_compressed_summary`（滚动摘要）：[agent_context.py:L23-L45](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/agent_context.py#L23-L45)
- 组装给模型的消息时，支持把摘要作为一条前置消息注入：`get_memory(prepend_summary=True)`：[agent_context.py:L138-L169](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/agent_context.py#L138-L169)

这里的关键设计点：

- 摘要不是“替换系统提示”，而是“作为一条消息前置”
- 过滤被标记为 COMPRESSED 的消息，避免重复注入

#### 4.2 约束：token 预算是稀缺资源

QwenPaw 把压缩触发建立在 token 计数上，并把系统提示 + 摘要 token 一并纳入预算：  
`pre_reasoning()` 计算 `sys_prompt` + `compressed_summary` 的 token，再决定还能容纳多少历史：[light_context_manager.py:L731-L777](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L731-L777)

这对应 Claude Code 的核心洞察：可用 token ≠ max token，必须主动管理。

---

### 第 2 层：External（模型外部长期记忆）

#### 4.3 后端接口：BaseMemoryManager（可插拔 + 后台队列）

QwenPaw 把长期记忆抽象为可注入后端：

- summarize/retrieve/dream 是可选能力（默认空实现），便于按需实现不同后端：[base_memory_manager.py:L78-L129](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/base_memory_manager.py#L78-L129)
- 提供后台 summarize 队列：`add_summarize_task()` 把任务串行 FIFO 处理，避免阻塞主链路：[base_memory_manager.py:L130-L182](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/base_memory_manager.py#L130-L182)

这回答了“怎么管”的一个工程细节：**把昂贵操作移出用户主路径**。

#### 4.4 默认实现：ReMeLightMemoryManager（文件记忆 + 语义检索注入）

QwenPaw 的一个核心落点是 `memory_search()`：

- 明确语义搜索目标：`MEMORY.md` 与 `memory/*.md`：[reme_light_memory_manager.py:L340-L389](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/reme_light_memory_manager.py#L340-L389)
- 支持 CJK 字符级分词（提升中文检索召回）：[reme_light_memory_manager.py:L300-L338](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/reme_light_memory_manager.py#L300-L338)

更关键的是“检索结果如何进入模型上下文”：

- `retrieve()` 会从最近消息拼 query（约 100 字符预算），调用 `memory_search()`，然后构造一对消息块：
  - assistant: tool_use(memory_search)
  - system: tool_result(memory_search)
- 最终把这两条追加到 `msg` 中，成为模型可见上下文的一部分：[reme_light_memory_manager.py:L415-L521](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/reme_light_memory_manager.py#L415-L521)

这对应“四问”里的“怎么取”：检索不是“凭空出现在系统提示”，而是 **明确的注入机制**，并且可被工具治理与观测（你能看到 tool_use/tool_result）。

---

### 第 3 层：Episodic（情景/经历记录）

QwenPaw 有一个很容易被忽视但很重要的雏形：**对话归档（dialog jsonl）**。

- 当历史消息被压缩或清空时，会落盘到 `dialog/YYYY-mm-dd.jsonl`：`_append_messages_to_dialog()`：[agent_context.py:L46-L136](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/agent_context.py#L46-L136)
- `mark_messages_compressed()` 会先落盘，再从内存中移除这些消息：[agent_context.py:L209-L246](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/agent_context.py#L209-L246)

它更接近“事件流水账”（原始经历），但目前并没有看到“按需检索 dialog 并注入”的主路径（也就是 Hermes 的 session_search 那条链路）。这给你一个非常明确的设计练习：**把 episodic archive 接入检索工具**。

---

### 第 4 层：Parametric（模型参数）

QwenPaw 不在运行时修改模型权重；因此 parametric memory 主要来自底座模型本身。  
工程上要做的是：避免把“应该落入 external 的稳定事实”误当作 parametric（否则会出现幻觉/遗忘）。

---

## 五、按“四个架构问题”再拆一遍 QwenPaw（把策略讲透）

### 5.1 存什么（Write Policy）

QwenPaw 的“写入”分两条路：

1) **写入滚动摘要（in-context 的压缩摘要）**  
- 触发：context compaction 成功时更新 `compressed_summary`：[light_context_manager.py:L922-L934](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L922-L934)
- 目的：让模型保持“近期 + 摘要”的上下文窗口可用

2) **写入长期记忆文件（external）**  
- 触发 A：压缩发生且 `summarize_when_compact=True`，入队 summarize：[light_context_manager.py:L927-L931](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L927-L931)
- 触发 B：按 `auto_memory_interval` 周期性入队 summarize（每 N 次用户提问）：[light_context_manager.py:L976-L1019](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L976-L1019)
- 执行：后台队列串行处理 summarize：[base_memory_manager.py:L130-L182](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/base_memory_manager.py#L130-L182)

你在设计自己的系统时，建议把“写什么”显式化成规则：

- 用户稳定偏好（风格、输出格式、禁忌）
- 项目稳定事实（技术栈、目录约定、关键入口）
- 长期目标与约束（安全、合规、性能边界）
- 不写：一次性信息、明显临时的 token/密钥、可从外部再获取的高噪声日志

### 5.2 存在哪（Store Policy）

QwenPaw 至少有三种介质：

- **会话状态文件（sessions/*.json）**：用于续聊回放（短中期、结构化）  
  - 保存：`save_session_state(..., agent=agent)`：[runner.py:L760-L766](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L760-L766)
  - 载入：`load_session_state(..., agent=agent)` 并 `rebuild_sys_prompt()`：[runner.py:L638-L656](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L638-L656)
  - 文件名安全与损坏恢复：SafeJSONSession + `_safe_json_loads`：[session.py:L24-L61](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/session.py#L24-L61)、[session.py:L97-L176](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/session.py#L97-L176)
- **长期记忆文件（MEMORY.md、memory/*.md）**：可读、可编辑、可检索（长期、可治理）  
  - 检索入口：`memory_search()`：[reme_light_memory_manager.py:L340-L389](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/reme_light_memory_manager.py#L340-L389)
- **对话归档（dialog/*.jsonl）**：原始经历沉淀（可复盘，但暂未强检索）  
  - 写入点：`mark_messages_compressed()`：[agent_context.py:L209-L246](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/agent_context.py#L209-L246)

这三种介质对应不同生命周期：会话续聊 / 长期偏好与事实 / 原始经历归档。

### 5.3 怎么取（Retrieve Policy）

QwenPaw 的“取”同样分两条：

1) **自动注入（推荐作为默认路径）**  
- 注入点：ContextManager 的 `pre_reply()`  
- 行为：在 reply 前调用 `memory_manager.retrieve()`，把检索结果以 tool_use/tool_result 的方式追加到 `msg`：[light_context_manager.py:L659-L708](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L659-L708)
- 关键细节：命令（/help /clear 等）会跳过检索，避免浪费 token 或干扰命令路径：[light_context_manager.py:L680-L684](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L680-L684)

2) **手动工具调用（用于可控检索/调试）**  
MemoryManager 以工具形式暴露给 agent（接口约定）：[base_memory_manager.py:L66-L76](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/base_memory_manager.py#L66-L76)

你可以把 QwenPaw 这条链路理解成 Hermes 的“按需检索注入”，只不过触发者是系统 hook（自动），而不是完全由 agent 自己决定（手动）。

### 5.4 怎么管（Manage Policy）

QwenPaw 的“治理”不是一个点，是一组互补策略：

#### 5.4.1 上下文压缩（compaction）

- 触发点：`pre_reasoning()`（推理前），基于 token 阈值判断：[light_context_manager.py:L710-L782](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L710-L782)
- 成功路径：调用 compactor 生成摘要，更新 `compressed_summary`，并把被压缩消息归档后移除：[light_context_manager.py:L805-L934](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L805-L934)
- 失败兜底：压缩失败或被禁用时，不中止流程，而是“放宽保留窗口，尽量保留近期、丢更老”：[light_context_manager.py:L823-L920](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L823-L920)

这个兜底非常关键：工程上要追求“失败不致命”，否则记忆系统会变成稳定性风险源。

#### 5.4.2 工具结果裁剪（tool-result pruning）

- 触发点：`post_acting()`（每次工具执行后），对过大的 tool_result 做截断与落盘：[light_context_manager.py:L944-L967](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L944-L967)
- 目的：防止日志/长输出撑爆上下文窗口，同时保留“可追溯的外部引用”（落盘文件可被 read_file 续读）

这对应“Claude Code 的上下文工程”：不是把所有东西都塞给模型，而是让信息进入上下文的方式可控。

---

## 六、关键链路：QwenPaw 把“记忆系统”挂进 Agent 主循环的方式

### 6.1 Hook 注入点（最核心的结构性设计）

QwenPawAgent 在构建后会注册 ContextManager 的四个 hook：

- pre_reply：记忆检索注入
- pre_reasoning：上下文压缩
- post_acting：工具结果裁剪
- post_reply：周期性自动记忆

见：[react_agent.py:L412-L453](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/react_agent.py#L412-L453)

为什么这很重要？

- 它把“记忆治理”从业务代码里剥离出来，形成可插拔的策略层
- 你可以替换 context_manager/memory_manager，而不改主循环

### 6.2 会话状态回放（跨会话延续）

Runner 每次请求会：

- 构造 agent，并把 `memory_manager/context_manager` 注入：[runner.py:L547-L557](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L547-L557)
- 载入 session state，再重建 system prompt（避免用到保存时的旧 prompt）：[runner.py:L638-L656](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L638-L656)
- 请求结束保存 state：[runner.py:L760-L766](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/runner/runner.py#L760-L766)

这里回答了“存在哪/怎么管”的另一个关键：**记忆系统不仅是检索与写入，还包括状态一致性与可恢复性**。

---

## 七、把 QwenPaw 放回“三大框架哲学”的坐标系（你该学什么）

### 7.1 QwenPaw 像 OpenClaw 的地方

- 长期记忆落在文件（MEMORY.md、memory/*.md），天然具备可读、可编辑、可版本控制的优势
- summarize/dream 本质上都是“从原始对话/日志提炼精华”的过程，与 OpenClaw 的“日志→精华”一致

### 7.2 QwenPaw 像 Claude Code 的地方

- 记忆治理不是“多存”，而是“精确调度”：压缩、裁剪、按需注入
- token 阈值、reserve、失败兜底，把“预算”做成了一等公民

### 7.3 QwenPaw 像 Hermes 的地方（以及缺的那一块）

- 已经有了分层：会话状态 / 长期文件 / 对话归档 / 检索注入
- 但“情景记忆 = 可复用技能/流程沉淀，并渐进加载”还没有形成一条强主链路

如果你要把 QwenPaw 的记忆系统推向 Hermes 的方向，最自然的演进是：

- 把 dialog/*.jsonl 变成可检索的 episodic store（FTS 或向量）
- 把“多步任务的执行轨迹”沉淀为 workflow/skill 文档（像 Hermes Skills）
- 引入渐进式披露：默认只注入 skill 索引，必要时注入全文

---

## 八、动手练习（建议按顺序做）

### 练习 1：画出你自己的“四层记忆地图”

写一个表格，列出每层记忆：

- 入口（写入触发点）
- 存储介质与格式
- 检索触发点
- 注入位置
- 失败兜底

对照 QwenPaw 代码，把每个格子写到“能点到文件”的程度。

### 练习 2：验证 QwenPaw 的“自动检索注入”链路

目标：确认你能在消息流里观察到 memory_search 的 tool_use/tool_result。

你需要读懂两个点：

- `pre_reply()` 何时会调用 retrieve（命令会跳过）：[light_context_manager.py:L659-L708](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L659-L708)
- `retrieve()` 如何构造 tool_use/tool_result 并追加到 msg：[reme_light_memory_manager.py:L415-L521](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/memory/reme_light_memory_manager.py#L415-L521)

### 练习 3：做一次“压缩失败演练”

目标：理解“失败不致命”的工程意义。

读 `pre_reasoning()` 的 fallback 分支，回答：

- 什么情况下 compact_content 会为空？
- fallback 为什么用更大的 reserve？
- fallback 的代价是什么（会丢什么）？

代码入口：[light_context_manager.py:L823-L920](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/light_context_manager.py#L823-L920)

### 练习 4：把 episodic archive 接到检索里（设计题）

不要求你现在实现，但要求你给出设计：

- 你会用 FTS 还是向量？为什么？
- 召回的粒度是“整天”“整段”“单条消息”？
- 你会把 episodic 结果注入到 msg 的哪个位置？（像 memory_search 一样 tool_result，还是 system prompt？）

落盘位置见：[agent_context.py:L46-L136](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/agents/context/agent_context.py#L46-L136)

---

## 九、验收清单

- [ ] 能用“四个架构问题”完整描述 QwenPaw 的记忆系统（不看代码也能讲清）
- [ ] 能指出 QwenPaw 记忆系统的 5 个关键点：session 回放、pre_reply 检索注入、pre_reasoning 压缩、post_acting 裁剪、post_reply 周期性沉淀
- [ ] 能解释为什么“压缩失败要兜底”，以及兜底的代价
- [ ] 能提出一套自己的写入/检索/更新策略（至少包含 3 条可执行规则）

