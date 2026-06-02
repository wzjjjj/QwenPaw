# Module 10 / Lesson 05：测试与调试：让你能“大胆改、快速回归、不怕线上”

## 本课总览（资深工程师的底气来自可验证性）

你要成为“资深的大模型应用工程师”，一个关键分水岭是：

- 初级：能实现功能，但不敢改、不敢重构、线上靠祈祷
- 资深：能把不确定系统（LLM、外部工具、网络抖动）装进可控边界，用测试与可观测性把风险前置

QwenPaw 已经提供了比较完整的 pytest 体系（含异步测试、临时目录隔离、monkeypatch、契约/集成分层）。本课把这些能力组织成你可迁移的“工程方法”。

## 学习目标

- 能在项目里写 pytest-asyncio 异步测试，并避免“事件循环冲突/资源泄漏”
- 能用 tmp_path/TemporaryDirectory 隔离文件系统副作用
- 能用 monkeypatch/mock 隔离外部依赖（provider、secret dir、网络）
- 能把日志与异常当成接口：定义错误语义、定位问题证据、写回归用例

## 先修知识

- 了解 pytest 基础（fixture、assert）
- 了解 asyncio 基础（来自 Lesson 01）

## 本节要回答的关键问题

1. 为什么异步测试要用 `pytest.mark.asyncio`？它和手写 event loop 的区别是什么？
2. 如何让测试不污染本机环境（working_dir/secret_dir/临时文件）？
3. monkeypatch 的正确用法是什么？它为什么适合测试配置与全局常量？
4. 当你修一个线上 bug，如何把“修复”变成“可回归的用例”？

## 核心概念

### 1) 测试分层：unit / contract / integration

QwenPaw 在 pytest 配置里明确了测试目录与 marker 分层：

- [pyproject.toml](file:///d:/编程学习记录/QwenPaw/pyproject.toml#L111-L123)

建议你按同样方式组织自己的 Agent 项目：

- unit：纯逻辑、无外部依赖、快
- contract：接口契约（输入/输出 shape、错误语义）、可模拟外部系统
- integration：最小链路联通（可能慢，但覆盖真实组合风险）

### 2) 异步测试：用 pytest-asyncio 管理 event loop

一个最小异步测试长这样：

- [test_workspace_creation](file:///d:/编程学习记录/QwenPaw/tests/unit/workspace/test_workspace.py#L8-L24)

你要注意两点：

- 不要在测试里自己 `asyncio.new_event_loop().run_until_complete(...)`，除非你真的在测“跨线程 event loop”
- 确保资源关闭：临时目录、后台 task、子进程、网络 client

### 3) 隔离副作用：tmp_path / TemporaryDirectory / monkeypatch

QwenPaw 的 provider manager 测试展示了一个非常实用的模式：把全局目录常量 monkeypatch 到临时目录，避免写到真实 secret 目录：

- [isolated_secret_dir fixture](file:///d:/编程学习记录/QwenPaw/tests/unit/providers/test_provider_manager.py#L84-L88)

这类测试能力对 Agent 工程尤其重要，因为你经常需要：

- 伪造 provider 配置与 secret 存储
- 伪造 workspace 目录与技能目录
- 伪造网络响应/超时/错误码

### 4) 调试策略：把“猜”变成“证据驱动”

工程里调试的核心不是“多打 print”，而是：

- 错误语义清晰：输入错误、权限错误、外部服务错误、超时错误要可区分
- 日志落点可追踪：能串起一次请求/一次 tool call/一次 channel 消费
- 回归用例固化：每次修复都应该能被测试复现

本项目里你可以优先学习的调试载体：

- 日志与命令行入口： [main.py](file:///d:/编程学习记录/QwenPaw/src/qwenpaw/cli/main.py#L31-L56)
- 典型的复杂测试基建： [test_provider_manager.py](file:///d:/编程学习记录/QwenPaw/tests/unit/providers/test_provider_manager.py#L84-L120)

## 代码走读路线（建议按这个顺序）

- pytest 配置与 marker 分层  
  - [pyproject.toml](file:///d:/编程学习记录/QwenPaw/pyproject.toml#L111-L123)
- 读一个“最小异步测试”模板  
  - [tests/unit/workspace/test_workspace.py](file:///d:/编程学习记录/QwenPaw/tests/unit/workspace/test_workspace.py#L8-L80)
- 读一个“monkeypatch 隔离副作用”模板  
  - [tests/unit/providers/test_provider_manager.py](file:///d:/编程学习记录/QwenPaw/tests/unit/providers/test_provider_manager.py#L84-L120)

## 动手练习

### 练习 1（基础）：为你改过的一处逻辑补齐最小 unit test

- 选择一个纯逻辑函数（不依赖网络/磁盘）
- 写 2 个断言：正常路径 + 一个失败路径（错误语义明确）

### 练习 2（进阶）：写一个异步测试，验证“取消/超时/清理”

- 选择 Lesson 01 里的目标（例如 TaskTracker 或某个长等待协程）
- 写 pytest-asyncio 测试：触发取消或超时，断言任务结束且不泄漏资源

### 练习 3（挑战）：用 monkeypatch 模拟外部系统失败并固化回归

- 模拟一个 provider/network/secret 存储失败场景
- 断言系统给出稳定、可解释的错误语义（而不是吞错或抛一堆底层异常）

## 验收清单

- [ ] 我能解释项目的测试分层（unit/contract/integration）以及它们的目标边界
- [ ] 我能写 pytest-asyncio 的异步测试，并能避免 event loop 冲突与资源泄漏
- [ ] 我能用 tmp_path/TemporaryDirectory/monkeypatch 把副作用隔离到测试环境
- [ ] 我能把一次 bug 修复固化成回归用例，并能解释覆盖的失败模式

## 下一步建议（把能力变成“可迁移”）

- 每做一次功能改动，强制自己回答：失败语义是什么？怎么观测？怎么回归？
- 每两周挑一个“并发/超时/取消”类问题做一次复盘：问题链路、证据、修复点、回归用例
