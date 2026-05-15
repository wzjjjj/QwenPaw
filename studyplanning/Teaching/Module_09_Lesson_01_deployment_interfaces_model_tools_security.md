---
title: Module 09 Lesson 01 - 部署接口三件套：模型侧 / 工具侧 / 安全侧（算法工程师视角）
---

## 学习目标

- 说清楚 QwenPaw 在企业部署里“你要对接的接口”到底是哪三块：模型、工具、安全
- 能把推理服务接进来（provider / endpoint / key / 参数），并指定某个 agent 使用哪一个模型
- 能把 agent 的可执行能力收敛到白名单（只开放必要的 tool/MCP，关掉泛化高风险工具）
- 能把敏感动作纳入审批链路（tool guard + approval_level），并理解审批触发点

## 先修知识

- 知道 QwenPaw 的 “agent = workspace = 独立运行时” 基本结构（每个 workspace 有自己的 runner）
- 会使用 CLI 创建 agent（例如 `qwenpaw agents create ...`）

## 本节要回答的关键问题

- “模型侧”到底配置在哪里，配置会落盘到哪？怎么给某个 agent 选择默认模型？
- “工具侧”怎么做最小权限？工具是在哪里被注册/过滤的？MCP 又是什么入口？
- “安全侧”什么时候会触发审批？审批走什么 API/命令？怎么设成企业可控的默认策略？

## 核心概念（你只需要记住这 3 个接口面）

### 1) 模型侧：Provider（推理服务的抽象）

QwenPaw 把“推理服务”抽象成 Provider：它包含 `base_url`、`api_key`、可用模型列表、以及 `generate_kwargs`（推理参数）。

- Provider 数据结构定义在 [provider.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/providers/provider.py#L73-L152)
- Provider 的保存/加密落盘由 [provider_manager.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/providers/provider_manager.py#L1232-L1274) 负责（`api_key` 等敏感字段会加密写入 secret 目录）
- secret/working 目录的定义在 [constant.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/constant.py#L89-L112)

对算法工程师来说，这意味着你们的“推理服务接入”只需要把以下信息对齐即可：

- provider_id：你要用哪个 Provider（例如 openai/ollama/自建兼容服务）
- base_url：推理服务的内网地址
- api_key：鉴权信息（如果需要）
- model_id：具体模型名称
- generate_kwargs：温度、top_p、max_tokens 等推理参数

某个 agent 用哪个模型，由 agent 的 `active_model` 决定（provider_id + model）。

### 2) 工具侧：Tool / MCP（业务系统能力入口）

你们做“经营工作站”的核心不是把提示词写漂亮，而是把“拉数/口径/工单/群发”等能力做成可审计、可控的工具入口。

QwenPaw 里工具入口分两类：

- Built-in tools：框架自带的一组工具函数（读写文件、shell、浏览器、代理协作等）
  - 这些工具是否启用，是按 agent 配置决定的，并在创建 toolkit 时过滤，见 [react_agent.py:_create_toolkit](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/agents/react_agent.py#L217-L303)
  - 控制台/API 可以列出/切换工具开关，见 [routers/tools.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/app/routers/tools.py#L34-L126)
- MCP tools：把工具放到独立进程/服务里，通过 MCP 协议接入（适合企业内网系统、权限治理、审计落地）
  - workspace 启动时会创建 MCPClientManager 服务，见 [workspace.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/app/workspace/workspace.py#L218-L227)

对于企业场景，推荐的“工具侧原则”是：

- 能做成专用工具的，就不要给 `execute_shell_command` 这种泛化工具
- 工具入参要结构化（包含 `request_id / operator / scope / reason / dry_run` 等），工具输出要可审计（包含口径版本、数据来源、时间范围）
- 业务系统接入优先考虑 MCP（便于独立部署、鉴权、审计、限流）

### 3) 安全侧：Tool Guard + Approval（敏感动作强制关口）

QwenPaw 的“审批”不是你手写 if/else，而是通过 Tool Guard 在工具调用前拦截。

- 拦截与决策逻辑在 [tool_guard_mixin.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L160-L333)
- `approval_level` 决定策略（STRICT/SMART/AUTO/OFF），并影响是否需要审批，见 [tool_guard_mixin.py:_get_tool_execution_level](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L113-L132)
- 进入审批等待后会创建 pending request 并阻塞，见 [_acting_with_approval](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L392-L467)
- 审批入口既有 API 也有聊天命令：
  - API：见 [routers/approval.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/app/routers/approval.py)
  - 命令：`/approval approve|deny|list`，见 [approval_handler.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/app/runner/control_commands/approval_handler.py)

企业部署里建议至少用 SMART（低风险自动放行，中高风险走审批）。对“群发/工单/触达/变更配置”等动作，建议做到“默认需要审批”。

## 代码走读路线（建议按这个顺序看）

- Provider 配置与持久化：
  - [ProviderInfo](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/providers/provider.py#L73-L152)
  - [ProviderManager._save_provider](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/providers/provider_manager.py#L1232-L1256)
  - Provider API：见 [routers/providers.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/app/routers/providers.py#L166-L230)
- Tool 启用/禁用：
  - 工具注册过滤：[react_agent.py:_create_toolkit](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/agents/react_agent.py#L217-L303)
  - 工具开关 API：[routers/tools.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/app/routers/tools.py#L34-L126)
- 安全与审批：
  - 审批触发点：[tool_guard_mixin.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/agents/tool_guard_mixin.py#L160-L333)
  - 审批 API：[routers/approval.py](file:///d:/%E7%BC%96%E7%A8%8B%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/QwenPaw/src/qwenpaw/app/routers/approval.py)

## 动手练习（建议按顺序做一遍）

### 练习 1：创建一个“经营分析专用 agent”，并收敛工具能力

1. 创建 agent（只装你要的 skills）
2. 进入控制台的 Tools 页面（或调用 tools API），把与经营无关、泛化风险高的内置工具关掉（尤其是 `execute_shell_command`）
3. 验证：同一个问题在不同 agent 下，工具能力是否不同（对应 agent 的工具列表不同）

验收要点：能解释“为什么这个 agent 没有某个工具”（因为 toolkit 创建时按 agent_config.tools.builtin_tools 过滤）。

### 练习 2：把推理服务接入 Provider，并指定 agent 使用该模型

1. 在控制台 Providers 页面配置 provider 的 base_url 与 api_key
2. 选择一个模型并设置为该 agent 的 active_model（可以在创建 agent 时带 `--provider-id/--model-id`，也可以通过控制台/接口更新）
3. 验证：对话时模型请求是否走你的推理服务（从推理服务侧日志或网关监控确认）

### 练习 3：打开 SMART 审批策略，让“敏感工具调用”必须走审批

1. 把目标 agent 的 `approval_level` 设为 SMART
2. 触发一次高风险工具调用（例如文件写入、外部请求、或你们自定义的“群发/建单”工具）
3. 在控制台或对话里执行 `/approval list`，然后 `/approval approve` 或 `/approval deny`

验收要点：能解释“触发审批的原因来自 tool guard，而不是提示词硬编码”。

## 验收清单

- [ ] 我能指出 provider/base_url/api_key/generate_kwargs 在代码层的入口位置
- [ ] 我能解释 agent 选模型的机制（active_model = provider_id + model_id）
- [ ] 我能把一个 agent 的 built-in tools 收敛成白名单，并验证生效
- [ ] 我能解释 MCP 的定位：企业内网系统能力更适合做成 MCP 工具
- [ ] 我能把敏感动作纳入审批，并完成 approve/deny 的闭环

## 下一课预告

- 如果你们要做“经营工作站”，下一步需要把“拉数/口径/工单/群发”做成真正的企业工具接口（优先 MCP），并把审计字段与权限体系贯穿进去。
