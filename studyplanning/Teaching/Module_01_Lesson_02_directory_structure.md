# Module 01 - Lesson 02: 顶层目录结构与模块划分

## 学习目标

本节课结束后，您将能够：
- 分析项目的目录结构和组织方式
- 理解各个模块的职责和边界
- 掌握项目的整体架构层次

## 先修知识

- 完成 Lesson 1.1 的学习
- 了解基本的软件项目目录结构
- 了解 Python 项目的常见组织方式

## 本节要回答的关键问题

1. QwenPaw 的顶层目录结构是怎样的？
2. 各个模块的职责和边界是什么？
3. 项目的整体架构层次是如何划分的？
4. 前端和后端是如何组织的？
5. 技能和工具是如何管理的？

## 核心概念

### 目录结构概述

QwenPaw 采用清晰的目录结构，将前端和后端代码分离，同时对核心功能进行模块化组织。项目的顶层目录结构如下：

- **console/**: 前端代码，基于 React + TypeScript
- **deploy/**: 部署相关配置和脚本
- **scripts/**: 构建、打包和工具脚本
- **src/**: 后端代码，基于 Python
- **website/**: 官方网站源码

### 后端架构层次

后端代码位于 `src/qwenpaw/` 目录下，采用分层架构：

1. **核心代理层**：包含代理的核心实现，如 `agents/` 目录
2. **应用层**：包含应用级功能，如 `app/` 目录
3. **工具层**：包含各种工具实现，如 `agents/tools/` 目录
4. **技能层**：包含内置和自定义技能，如 `agents/skills/` 目录
5. **内存层**：包含内存和上下文管理，如 `agents/memory/` 目录

### 前端架构

前端代码位于 `console/` 目录下，基于 React + TypeScript：

1. **src/api/**: API 客户端代码
2. **src/components/**: 前端组件
3. **src/pages/**: 页面组件
4. **src/stores/**: 状态管理
5. **src/utils/**: 工具函数

## 代码走读路线

1. **项目根目录** - 了解整体结构
2. **src/qwenpaw/** - 后端核心代码
3. **console/src/** - 前端核心代码
4. **src/qwenpaw/agents/** - 代理系统实现

## 关键代码讲解

### 后端核心目录结构

```
src/qwenpaw/
├── agent_stats/       # 代理统计信息
├── agents/            # 代理核心实现
│   ├── acp/           # 代理控制协议
│   ├── context/       # 上下文管理
│   ├── hooks/         # 钩子系统
│   ├── md_files/      #  markdown 文件
│   ├── memory/        # 内存管理
│   ├── mission/       # 任务管理
│   ├── skills/        # 技能系统
│   ├── tools/         # 工具系统
│   └── utils/         # 工具函数
├── app/               # 应用级功能
├── __init__.py        # 包初始化
├── __main__.py        # 主入口
└── __version__.py     # 版本信息
```

### 前端核心目录结构

```
console/src/
├── api/              # API 客户端
├── components/       # 通用组件
├── constants/        # 常量定义
├── contexts/         # 上下文
├── hooks/            # 自定义钩子
├── layouts/          # 布局组件
├── locales/          # 国际化
├── pages/            # 页面组件
├── plugins/          # 插件系统
├── stores/           # 状态管理
├── styles/           # 样式文件
├── utils/            # 工具函数
├── App.tsx           # 应用入口
├── i18n.ts           # 国际化配置
└── main.tsx          # 主入口
```

### 代理系统核心文件

`src/qwenpaw/agents/__init__.py` 定义了代理系统的基本结构：

```python
# 代理系统初始化
```

`src/qwenpaw/react_agent.py` 实现了基于 React 模式的代理：

```python
# React 代理实现
```

## 动手练习

1. **探索目录结构**：使用命令行或文件浏览器浏览项目的目录结构
   - 记录关键目录的作用
   - 分析目录结构的设计意图

2. **查看核心文件**：打开并阅读以下核心文件
   - `src/qwenpaw/__main__.py`
   - `src/qwenpaw/agents/__init__.py`
   - `console/src/App.tsx`

3. **绘制架构图**：根据目录结构，绘制 QwenPaw 的整体架构图
   - 标注各个模块的职责
   - 展示模块间的依赖关系

## 验收清单

- [ ] 理解项目的顶层目录结构
- [ ] 掌握各个模块的职责和边界
- [ ] 了解项目的整体架构层次
- [ ] 分析前端和后端的组织方式
- [ ] 绘制 QwenPaw 的架构图

## 下一课预告

下一课我们将分析 QwenPaw 的关键技术选型与依赖，了解项目的技术栈选择和核心依赖库的作用。