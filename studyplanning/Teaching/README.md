# QwenPaw 源码学习指南

## 课程简介

欢迎来到 QwenPaw 源码学习课程！本课程旨在帮助您从零开始，逐层深入地理解 QwenPaw 项目的架构设计、核心功能和实现原理。

QwenPaw 是一个功能强大的个人 AI 助手，具有以下核心特性：
- **完全可控**：内存和个性化完全由您控制，可本地部署或云部署
- **技能扩展**：内置调度、PDF/Office 处理、新闻摘要等功能，支持自定义技能
- **多代理协作**：创建多个独立代理，支持代理间通信协作
- **多层安全**：工具防护、文件访问控制、技能安全扫描
- **多渠道支持**：钉钉、飞书、微信、Discord、Telegram 等
- **记忆进化与主动服务**：从交互中学习，主动为您服务

## 模块目录

本课程分为以下六个模块，从宏观到微观，循序渐进地讲解 QwenPaw 的源码结构：

### 模块 1：全局架构与设计理念
- [Lesson 1.1: 项目目标与核心价值](Module_01_Lesson_01_project_goals.md)
- [Lesson 1.2: 顶层目录结构与模块划分](Module_01_Lesson_02_directory_structure.md)
- [Lesson 1.3: 关键技术选型与依赖](Module_01_Lesson_03_tech_stack.md)

### 模块 2：核心代理系统
- [Lesson 2.1: 代理架构与生命周期](Module_02_Lesson_01_agent_architecture.md)
- [Lesson 2.2: 模型管理与调用](Module_02_Lesson_02_model_management.md)
- [Lesson 2.3: 命令处理与路由](Module_02_Lesson_03_command_handling.md)

### 模块 3：技能系统
- [Lesson 3.1: 技能架构与加载机制](Module_03_Lesson_01_skill_architecture.md)
- [Lesson 3.2: 内置技能分析](Module_03_Lesson_02_builtin_skills.md)
- [Lesson 3.3: 自定义技能开发](Module_03_Lesson_03_custom_skills.md)

### 模块 4：内存与上下文管理
- [Lesson 4.1: 内存系统架构](Module_04_Lesson_01_memory_architecture.md)
- [Lesson 4.2: 上下文管理机制](Module_04_Lesson_02_context_management.md)
- [Lesson 4.3: 记忆进化与主动服务](Module_04_Lesson_03_memory_evolving.md)

### 模块 5：多渠道集成
- [Lesson 5.1: 渠道架构与扩展](Module_05_Lesson_01_channel_architecture.md)
- [Lesson 5.2: 内置渠道实现](Module_05_Lesson_02_builtin_channels.md)
- [Lesson 5.3: 渠道消息处理](Module_05_Lesson_03_message_processing.md)

### 模块 6：安全机制
- [Lesson 6.1: 安全架构与防护层](Module_06_Lesson_01_security_architecture.md)
- [Lesson 6.2: 工具防护与文件访问控制](Module_06_Lesson_02_tool_guard.md)
- [Lesson 6.3: 技能安全扫描](Module_06_Lesson_03_skill_security.md)

## 学习建议

1. **循序渐进**：建议按照模块顺序学习，从全局架构开始，逐步深入到具体功能实现
2. **理论与实践结合**：每节课都包含动手练习，建议实际操作以加深理解
3. **代码走读**：跟随课程中的代码走读路线，实际查看源码文件
4. **问题导向**：每节课都提出了关键问题，带着问题学习会更有针对性
5. **实验验证**：通过破坏性实验等方式验证系统的设计意图和边界情况

## 环境准备

为了更好地学习 QwenPaw 源码，建议您：

1. 克隆 QwenPaw 仓库：`git clone https://github.com/agentscope-ai/QwenPaw.git`
2. 安装依赖：参考项目 README.md 中的安装指南
3. 启动应用：运行 `qwenpaw init --defaults` 和 `qwenpaw app`
4. 打开控制台：访问 http://127.0.0.1:8088/ 查看界面

## 学习资源

- [QwenPaw 官方文档](https://qwenpaw.agentscope.io/)
- [GitHub 仓库](https://github.com/agentscope-ai/QwenPaw)
- [AgentScope 项目](https://github.com/agentscope-ai/agentscope)

祝您学习愉快！