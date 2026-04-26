# Module 03 - Lesson 02: 内置技能分析

## 学习目标

本节课结束后，您将能够：
- 了解内置技能的分类和功能
- 分析核心技能的实现原理
- 掌握技能的配置和使用方式

## 先修知识

- 完成 Lesson 3.1 的学习
- 了解基本的技能概念
- 了解常见的办公自动化和信息处理任务

## 本节要回答的关键问题

1. QwenPaw 有哪些内置技能？
2. 内置技能如何分类？
3. 核心技能的实现原理是什么？
4. 如何配置和使用内置技能？
5. 内置技能的适用场景是什么？

## 核心概念

### 内置技能分类

QwenPaw 的内置技能可以分为以下几类：

1. **办公自动化**：
   - docx：Word 文档处理
   - pptx：PowerPoint 演示文稿处理
   - xlsx：Excel 电子表格处理
   - pdf：PDF 文档处理

2. **信息获取**：
   - news：新闻摘要
   - himalaya：喜马拉雅音频处理

3. **工具集成**：
   - browser_cdp：浏览器控制（CDP 协议）
   - browser_visible：可视化浏览器控制
   - file_reader：文件读取

4. **任务调度**：
   - cron：定时任务调度

5. **代理协作**：
   - chat_with_agent：代理间通信
   - channel_message：渠道消息处理

### 技能实现原理

内置技能的实现原理包括：

1. **技能定义**：通过 SKILL.md 文件定义技能的元数据和参数
2. **技能实现**：通过脚本文件实现技能的核心逻辑
3. **技能注册**：通过技能管理器注册技能
4. **技能执行**：通过技能执行环境执行技能

### 技能配置与使用

技能的配置与使用包括：

1. **技能配置**：通过控制台或配置文件配置技能参数
2. **技能触发**：通过用户命令或其他技能触发技能执行
3. **技能参数**：根据技能要求提供必要的参数
4. **技能结果**：处理技能执行的结果

## 代码走读路线

1. **src/qwenpaw/agents/skills/** - 内置技能目录
2. **src/qwenpaw/agents/skills/pdf-en/SKILL.md** - PDF 技能定义
3. **src/qwenpaw/agents/skills/docx-en/scripts/** - Word 技能实现
4. **src/qwenpaw/agents/skills/cron-en/SKILL.md** - 定时任务技能定义

## 关键代码讲解

### PDF 技能定义

`src/qwenpaw/agents/skills/pdf-en/SKILL.md` 定义了 PDF 技能：

```markdown
# PDF 技能定义
```

### Word 技能实现

`src/qwenpaw/agents/skills/docx-en/scripts/` 目录包含了 Word 技能的实现：

```python
# Word 技能实现
```

### 定时任务技能定义

`src/qwenpaw/agents/skills/cron-en/SKILL.md` 定义了定时任务技能：

```markdown
# 定时任务技能定义
```

## 动手练习

1. **测试 PDF 技能**：使用 PDF 技能处理 PDF 文档
   - 上传 PDF 文档
   - 测试文档摘要功能
   - 测试文档内容提取

2. **测试 Word 技能**：使用 Word 技能处理 Word 文档
   - 上传 Word 文档
   - 测试文档编辑功能
   - 测试文档转换功能

3. **测试定时任务技能**：使用定时任务技能设置定时任务
   - 创建定时任务
   - 测试任务执行
   - 分析任务执行结果

## 验收清单

- [ ] 了解内置技能的分类和功能
- [ ] 分析核心技能的实现原理
- [ ] 掌握技能的配置和使用方式
- [ ] 测试 PDF 技能的功能
- [ ] 测试 Word 技能的功能
- [ ] 测试定时任务技能的功能

## 下一课预告

下一课我们将学习如何开发自定义技能，了解自定义技能的开发规范、打包和部署方法。