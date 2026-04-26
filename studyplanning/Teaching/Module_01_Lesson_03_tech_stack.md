# Module 01 - Lesson 03: 关键技术选型与依赖

## 学习目标

本节课结束后，您将能够：
- 分析项目的技术栈选择
- 了解核心依赖库的作用
- 掌握项目的构建和部署方式

## 先修知识

- 完成 Lesson 1.2 的学习
- 了解基本的 Python 和前端技术栈
- 了解常见的 AI 相关库

## 本节要回答的关键问题

1. QwenPaw 使用了哪些主要技术栈？
2. 核心依赖库的作用是什么？
3. 项目的构建流程是怎样的？
4. 部署方式有哪些？
5. 如何进行本地开发和测试？

## 核心概念

### 技术栈概述

QwenPaw 采用了现代化的技术栈，包括：

**后端**：
- Python 3.10+：主要开发语言
- FastAPI：Web 框架
- Pydantic：数据验证
- SQLAlchemy：ORM（可能使用）
- Agentscope：代理框架

**前端**：
- React：前端框架
- TypeScript：类型系统
- Vite：构建工具
- Ant Design：UI 组件库

**部署**：
- Docker：容器化
- pip：Python 包管理
- npm：前端包管理

### 核心依赖

1. **Agentscope**：QwenPaw 的核心依赖，提供代理框架和基础功能
2. **FastAPI**：提供高性能的 Web API
3. **React**：前端界面构建
4. **TypeScript**：前端类型安全
5. **Vite**：前端构建工具
6. **Docker**：容器化部署

### 构建与部署流程

1. **前端构建**：在 `console` 目录下执行 `npm ci && npm run build`
2. **后端安装**：使用 `pip install -e .` 安装 Python 包
3. **部署方式**：
   - 本地部署：直接运行 `qwenpaw app`
   - Docker 部署：使用 Docker 容器
   - 云部署：如阿里云 ECS

## 代码走读路线

1. **pyproject.toml** - Python 项目配置和依赖
2. **console/package.json** - 前端项目配置和依赖
3. **docker-compose.yml** - Docker 部署配置
4. **scripts/** - 构建和部署脚本

## 关键代码讲解

### Python 依赖配置

`pyproject.toml` 文件定义了项目的 Python 依赖：

```toml
# Python 依赖配置
```

### 前端依赖配置

`console/package.json` 文件定义了前端项目的依赖：

```json
# 前端依赖配置
```

### Docker 部署配置

`docker-compose.yml` 文件定义了 Docker 部署配置：

```yaml
# Docker 部署配置
```

### 构建脚本

`scripts/build_common.py` 包含了构建相关的通用功能：

```python
# 构建脚本
```

## 动手练习

1. **查看依赖配置**：打开并分析以下配置文件
   - `pyproject.toml`
   - `console/package.json`
   - `docker-compose.yml`

2. **尝试构建项目**：按照 README.md 中的指南构建项目
   - 构建前端：`cd console && npm ci && npm run build`
   - 安装后端：`pip install -e .`

3. **尝试不同部署方式**：
   - 本地部署：`qwenpaw init --defaults && qwenpaw app`
   - Docker 部署：`docker-compose up`

## 验收清单

- [ ] 理解项目的技术栈选择
- [ ] 掌握核心依赖库的作用
- [ ] 了解项目的构建流程
- [ ] 尝试不同的部署方式
- [ ] 分析部署配置文件

## 下一课预告

下一课我们将开始学习模块 2：核心代理系统，首先了解代理的架构与生命周期管理。