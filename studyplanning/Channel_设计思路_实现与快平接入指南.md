# Channel 设计思路、实现机制与“快平”接入指南（QwenPaw）

面向场景：你需要向领导解释 QwenPaw 的 channel 设计思路，并落地接入平安银行自研通讯 App（快平）。本文从“为什么这么设计 → 代码怎么落地 → 新增一个 channel 怎么做”三层拆解。

---

## 1. Channel 在整体架构里的定位（设计思路）

### 1.1 目标：把“消息平台差异”隔离在边界层

QwenPaw 的核心能力（LLM + agent loop + tools/mcp + memory）不应该耦合任何具体 IM 平台（钉钉/飞书/微信/企业微信/QQ/Matrix/…）。因此它把“对外通信”抽象为 channel：

- 对外：接入不同平台的接收方式（Webhook / WebSocket / 长轮询 / SDK 回调 / MQ 等）
- 对内：统一转成 AgentRequest 并调用同一条处理链（runner.stream_query）
- 对外回写：把 agent 输出的“消息内容/图片/文件”等，转换回平台 API 调用

换句话说：channel = Adapter/Anti-Corruption Layer，把平台协议适配成统一的运行时协议（AgentRequest/Events）。

### 1.2 设计关键点（你和领导沟通时可以重点讲）

1) **统一入口（process handler）**  
所有 channel 的业务处理统一走 `process(request) -> AsyncIterator[Event]`，由 runner 提供：  
见 [utils.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/utils.py#L139-L153)

2) **统一调度（ChannelManager + per-session 队列）**  
平台侧消息通常是并发到达、并且同一个会话要串行（防止上下文乱序）。QwenPaw 用 `ChannelManager` 把消息按 `(channel, session_id, priority)` 分流入独立队列消费，并带“批量合并”能力（例如快速连发图片+文字）。  
见 [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L68-L541)

3) **统一协议（BaseChannel 负责 inbound/outbound 的最小契约）**  
每个 channel 需要实现“native payload -> AgentRequest”的转换，以及“发送文本/附件”的落地；公共逻辑（debounce、allowlist、工具输出渲染、TaskTracker 对接）放在 `BaseChannel`。  
见 [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L78-L1246)

4) **可插拔（registry 支持自定义 channel 模块）**  
内置 channel 通过内置 registry 懒加载；自定义 channel 则从 `WORKING_DIR/custom_channels/` 动态发现并注册。  
见 [registry.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/registry.py#L20-L195)、以及常量 [constant.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/constant.py#L216-L219)

5) **配置可扩展（ChannelConfig 允许 extra keys）**  
内置 channel 用 Pydantic 模型定义；自定义 channel 的配置键也允许写进 channels 配置里（作为 extra dict 被 `ChannelManager.from_config` 取出）。  
见 [config.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/config.py#L415-L436) 与 [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L113-L216)

---

## 2. 实现机制：从“收到快平消息”到“回快平消息”的链路

### 2.1 Inbound（平台消息进入 QwenPaw）

#### Step A：平台事件进入 channel

不同平台的接入方式不同，但最终都要走到同一个动作：**把一个 payload 入队**。

常见模式：
- WebSocket/SDK 回调线程：收到消息后 `self._enqueue(native_payload)`（如 WeCom/QQ 等）
- 长轮询线程：轮询到消息后 `self._enqueue(native_payload)`（如 Weixin）
- Webhook HTTP：在 FastAPI 路由里拿到事件后，定位到 channel/manager，再 enqueue

其中 enqueue 由 manager 在启动时注入：
- `ChannelManager.start_all()` 给每个 channel 调用 `set_enqueue(cb)`  
见 [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L462-L492) 与 [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L351-L354)

#### Step B：ChannelManager 负责会话串行 + 批量合并

`enqueue(channel_id, payload)` 是线程安全的：可以从同步线程/SDK 回调里调用，manager 会切回事件循环。  
见 [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L352-L364)

消费侧：
- 一个队列对应一个 `(channel_id, session_id, priority)` 键
- consumer loop 会 drain 同键队列，形成 batch，调用 `_process_batch()` 合并处理  
见 [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L365-L461) 与 [_process_batch](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L39-L66)

#### Step C：BaseChannel 把 payload 转成 AgentRequest，并调用 runner.process

核心转换点是 `build_agent_request_from_native()`：子类必须实现。  
见 [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L642-L668)

随后 BaseChannel 会进入 `_run_process_loop`，对 Event 流做统一处理：
- `content` 事件（常见为工具输出流）可选渲染
- `message` completed 事件：转成可发送的 content parts，并调用 `send_content_parts`
见 [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L865-L1181)

> 你可以把它理解成：channel 负责把“输入适配”成 AgentRequest，再把“输出适配”回平台。

### 2.2 Outbound（把 agent 输出回写到平台）

BaseChannel 默认输出路径：

1) `on_event_message_completed()` 默认调用 `send_message_content()`  
见 [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L978-L990)

2) `send_message_content()` 会把 runtime Message 转成 `TextContent/ImageContent/FileContent/...` 等 parts  
见 [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L1041-L1077)

3) `send_content_parts()` 默认把文本合并成一个 body，并把媒体 URL 以 fallback 形式拼进文本，然后调用 `send()`  
见 [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L1129-L1181)

4) 子类可覆盖：
- `send()`：最关键，真正调用平台 API 发消息
- `send_media()`：真正上传/发送附件（图片/文件/音频/视频）
- `on_event_message_completed()`：平台有“卡片消息/合并消息/一次 API 只能发一条”等限制时，在此做批处理

---

## 3. Channel 的“契约”：一个 channel 至少要实现什么

从 QwenPaw 的实现看，新增 channel 的最小可用集是：

1) 类属性 `channel`：该 channel 的 key（字符串）
2) 构造：`__init__(process, enabled, ... )`（保存连接信息、token、前缀等）
3) `from_config(process, config, ...)`：从配置创建实例（建议同时支持 dict config）
4) `start()` / `stop()`：启动/停止接收环（线程、ws、poll 等）
5) `build_agent_request_from_native(native_payload)`：native -> AgentRequest（核心）
6) `send(to_handle, text, meta)`：把文本发回平台（核心）
7) （可选）`send_media(to_handle, part, meta)`：把附件发回平台
8) （可选）`resolve_session_id(sender_id, meta)`：定义会话粒度（单聊/群聊/线程等）

其中 5) 与 6) 决定了“能不能跑通闭环”。

---

## 4. 怎么接入新的 channel：以“快平”作为自定义 channel 的落地方式

### 4.1 两种接入形态（先选你们快平更接近哪种）

#### 形态 1：快平 Push 到你（Webhook / MQ / 反向 WS）

特点：快平侧主动把消息事件推给你。你需要：
- 在 QwenPaw 的 FastAPI 上暴露 `/api/kuaip/...` 接口接收事件
- 校验签名/证书/来源 IP
- 解析成 native payload，入队

QwenPaw 支持 custom channel 注册 FastAPI 路由：  
见 [registry.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/registry.py#L136-L189)

#### 形态 2：你 Pull 快平（长轮询 / SDK / WS 主动连接）

特点：你在 `start()` 里起线程/协程，持续拉取快平消息。你需要：
- 实现 `start()` 创建 poll/ws loop（可用后台线程）
- 收到消息后调用 `self._enqueue(native_payload)`

可类比现有实现：
- Weixin 长轮询线程：见 [weixin/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py)
- WeCom WebSocket：见 [wecom/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/wecom/channel.py)

### 4.2 快平 channel 的“数据模型”怎么映射

你需要在快平与 QwenPaw 之间建立三类映射：

1) **身份映射**（who）
- `sender_id`：快平用户唯一标识（员工号/unionId/openId 等）
- `user_id`：QwenPaw 内部把它当成 user；通常等于 sender_id

2) **会话映射**（where / which conversation）
- 单聊：`session_id = kuaip:<sender_id>`（一人一会话）
- 群聊：`session_id = kuaip:group:<group_id>`（一群一会话）
- 线程/话题：`session_id = kuaip:thread:<thread_id>`（一线程一会话）

对应扩展点：`resolve_session_id(sender_id, meta)`  
见 [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L595-L606)

3) **内容映射**（what）

QwenPaw 期望输入内容是 runtime content parts（TextContent/ImageContent/FileContent/...）。子类在 `build_agent_request_from_native` 里把快平原始消息解析成 parts 列表，再调用 `build_agent_request_from_user_content()` 拼成 AgentRequest。  
见 [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L607-L640)

### 4.3 最小实现骨架（快平）

你们落地时基本就是写一个自定义模块，放到 `WORKING_DIR/custom_channels/`，并在 agent 的 channels 配置里启用它。

下面是骨架（示意）：

```python
from __future__ import annotations

from typing import Any, Dict, List, Optional

from agentscope_runtime.engine.schemas.agent_schemas import (
    ContentType,
    TextContent,
)

from qwenpaw.app.channels.base import BaseChannel


class KuaiPingChannel(BaseChannel):
    channel = "kuaip"

    def __init__(
        self,
        process,
        enabled: bool,
        api_base: str,
        token: str,
        bot_prefix: str = "",
        on_reply_sent=None,
        show_tool_details: bool = True,
        filter_tool_messages: bool = False,
        filter_thinking: bool = False,
        **kwargs,
    ):
        super().__init__(
            process,
            on_reply_sent=on_reply_sent,
            show_tool_details=show_tool_details,
            filter_tool_messages=filter_tool_messages,
            filter_thinking=filter_thinking,
        )
        self.enabled = enabled
        self.api_base = api_base.rstrip("/")
        self.token = token
        self.bot_prefix = bot_prefix or ""

    @classmethod
    def from_config(
        cls,
        process,
        config,
        on_reply_sent=None,
        show_tool_details: bool = True,
        filter_tool_messages: bool = False,
        filter_thinking: bool = False,
        workspace_dir=None,
    ):
        c = config if isinstance(config, dict) else getattr(config, "__dict__", {})
        return cls(
            process=process,
            enabled=bool(getattr(config, "enabled", None) or c.get("enabled", False)),
            api_base=c.get("api_base", ""),
            token=c.get("token", ""),
            bot_prefix=c.get("bot_prefix", "") or "",
            on_reply_sent=on_reply_sent,
            show_tool_details=show_tool_details,
            filter_tool_messages=filter_tool_messages,
            filter_thinking=filter_thinking,
            workspace_dir=workspace_dir,
        )

    async def start(self) -> None:
        if not self.enabled:
            return
        # 形态 1：如果你用 webhook，则 start 可能是 no-op
        # 形态 2：如果你要主动拉取/WS，这里启动后台线程/协程

    async def stop(self) -> None:
        # 停止后台线程/关闭连接
        return

    def build_agent_request_from_native(self, native_payload: Any):
        payload = native_payload if isinstance(native_payload, dict) else {}
        sender_id = payload.get("sender_id") or ""
        meta = payload.get("meta") or {}

        session_id = payload.get("session_id") or self.resolve_session_id(
            sender_id,
            meta,
        )
        text = payload.get("text") or ""
        content_parts: List[Any] = [
            TextContent(type=ContentType.TEXT, text=text or " "),
        ]

        req = self.build_agent_request_from_user_content(
            channel_id=self.channel,
            sender_id=sender_id,
            session_id=session_id,
            content_parts=content_parts,
            channel_meta=meta,
        )
        setattr(req, "channel_meta", dict(meta))
        return req

    async def send(self, to_handle: str, text: str, meta: Optional[Dict[str, Any]] = None):
        # 调用快平发消息 API：to_handle 可以设计成 user_id / chat_id
        # 建议：把快平的 reply target 放进 meta，例如 meta["chat_id"]
        raise NotImplementedError
```

这段骨架的关键点：
- inbound 用 native dict（你自己定义结构），最终产出 AgentRequest
- outbound 把 `send(to_handle, text, meta)` 对接到快平发消息 API

### 4.4 自定义 channel 的注册与配置启用

#### 4.4.1 注册：放进 custom_channels 目录即可被发现

QwenPaw 会扫描 `WORKING_DIR/custom_channels/`：
- `.py` 文件或包目录（含 `__init__.py`）
- 找到其中所有 `BaseChannel` 子类
- 读取类属性 `channel` 作为 key 注册
见 [registry.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/registry.py#L98-L130)

#### 4.4.2 配置：在 channels 里增加 `kuaip` 配置块

配置结构由 `ChannelConfig` 定义，但它允许 extra，因此你可以写：
```json
{
  "channels": {
    "kuaip": {
      "enabled": true,
      "bot_prefix": "[QwenPaw]",
      "api_base": "https://kuaip.example/api",
      "token": "******",
      "filter_tool_messages": false,
      "filter_thinking": false
    }
  }
}
```

对应依据：
- `ChannelConfig` 允许 extra：见 [config.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/config.py#L415-L436)
- `ChannelManager.from_config` 会读取 extra 并把 dict 传入 `from_config`：见 [manager.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/manager.py#L131-L216)

#### 4.4.3 启停：enabled + 环境变量可进一步筛选

运行时还可以通过环境变量控制“允许哪些 channel 生效”：
- `QWENPAW_ENABLED_CHANNELS`：白名单
- `QWENPAW_DISABLED_CHANNELS`：黑名单
见 [utils.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/config/utils.py#L341-L371)

### 4.5 Webhook 形态下：如何把快平事件接入 FastAPI（关键机制）

QwenPaw 提供了一个约定：自定义 channel 模块可以在模块级别实现 `register_app_routes(app)`，由系统启动时调用，用来注册 `/api/...` 的路由。  
见 [registry.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/registry.py#L136-L189)

建议做法（思路）：

1) 在 `custom_channels/kuaip.py`（或包）里提供：
- `register_app_routes(app)`：注册 `/api/kuaip/webhook`  
2) webhook handler 中：
- 校验快平签名
- 从 `app.state.multi_agent_manager` 获取当前 agent/workspace（注意多 agent 场景）
- 拿到对应 workspace 的 `channel_manager`
- `channel_manager.enqueue("kuaip", native_payload)`

> 这里的关键不是“路由怎么写”，而是“最终入队”这一动作；入队后，后续链路统一走 ChannelManager/BaseChannel/Runner。

---

## 5. 快平接入时的工程化注意点（银行内网场景强相关）

### 5.1 安全与合规（强建议在评审里明确）

- Webhook 校验：签名、时间戳、防重放（message_id + TTL）
- 访问控制：IP allowlist / mTLS / 网关鉴权
- 数据最小化：meta 里避免塞敏感信息；日志脱敏（员工号/手机号/客户信息）
- Token 管理：不要写死在仓库；走配置或密钥管理系统

### 5.2 会话与幂等（常见坑）

- **session_id 设计决定“上下文隔离”**：群聊要不要共享上下文？线程消息要不要独立？先定规则再写 `resolve_session_id`
- 幂等：平台可能重复投递 webhook；建议用 `(platform_msg_id)` 做 dedup（可以存在 meta 里，或在 channel 内部做缓存）
- 合并：图片/文件和文字可能分开到达，BaseChannel 已提供“无文本先缓冲、文本到来再合并”的 debounce（尤其适合 IM 的附件上传行为）  
见 [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L281-L314) 与 [base.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/base.py#L764-L796)

### 5.3 对照现有 channel 的最佳实践（推荐你先读这几个）

- “存储会话级回调地址/用于主动发送”的设计：DingTalk 在 `_before_consume_process` 存储 webhook，供后续 cron/主动推送使用  
见 [dingtalk/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/dingtalk/channel.py#L332-L421)
- “消息入队 + 后台接收线程”的模式：WeCom/Weixin/QQ（不同协议但同一套路）  
见 [wecom/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/wecom/channel.py) / [weixin/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/weixin/channel.py) / [qq/channel.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/app/channels/qq/channel.py)

---

## 6. 你可以直接拿去汇报的“总结话术”

1) QwenPaw 把对外 IM 平台集成抽象成 channel，核心 agent 只认统一的 AgentRequest/Events。  
2) ChannelManager 按会话串行消费 + 优先级队列 + 批量合并，解决并发与乱序问题。  
3) BaseChannel 定义最小契约：native->AgentRequest、事件回写、debounce、策略控制与渲染。  
4) 自定义 channel 通过 `custom_channels/` 动态发现，配置通过 `channels.<key>`（extra dict）启用；如需 webhook，可用 `register_app_routes(app)` 挂载 `/api/` 路由。  
5) 快平接入的核心落点：确定“会话粒度(session_id)”与“回写目标(to_handle/meta)”的映射，然后实现 `build_agent_request_from_native` + `send` 即可跑通闭环。

