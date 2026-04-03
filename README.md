# Claude Code 源码深度分析白皮书

> 从入门到精通的完整技术指南

[![Claude Code](https://img.shields.io/badge/Claude-Code%202.1.31-blue.svg)](https://github.com/anthropics/claude-code)
[![License](https://img.shields.io/badge/license-反编译学习-orange.svg)](./LICENSE)
[![Stars](https://img.shields.io/github/stars/fysh1010/claude-code-whitepaper?style=flat)](https://github.com/fysh1010/claude-code-whitepaper/stargazers)

## 📚 文档简介

本白皮书是基于 Claude Code 2.1.31 版本反编译源码编写的完整技术文档，旨在帮助新手小白和技术人员快速学习掌握 Claude Code 源码和使用技巧。

> **注意**：本项目是反编译代码，仅供学习参考使用。使用前请评估相关法律风险。

## 📖 目录导航

### 基础入门篇

| 章节 | 标题 | 描述 |
|------|------|------|
| [01](./01-项目介绍.md) | 项目介绍 | 什么是 Claude Code 反编译项目，版本历史，项目目标 |
| [02](./02-架构概览.md) | 架构概览 | 整体目录结构、核心模块、技术栈、构建方式 |

### 核心原理篇

| 章节 | 标题 | 描述 |
|------|------|------|
| [03](./03-入口与启动.md) | 入口与启动 | cli.tsx → main.tsx → init.ts → setup.ts 完整启动流程 |
| [04](./04-查询引擎.md) | 查询引擎 | query.ts、QueryEngine.ts、API 客户端的协作机制 |
| [05](./05-工具系统.md) | 工具系统 | Tool.ts、工具注册表、工具执行模型 |
| [06](./06-UI与REPL.md) | UI与REPL | REPL 屏幕、消息渲染、输入处理、权限系统 |
| [07](./07-状态管理与MCP.md) | 状态管理与MCP | 状态管理架构、MCP 协议支持、权限系统 |

### 实战篇

| 章节 | 标题 | 描述 |
|------|------|------|
| [08](./08-工具实现示例.md) | 工具实现示例 | BashTool、FileEditTool、GrepTool 深度解析 |
| [09](./09-使用指南.md) | 使用指南 | 安装配置、交互方式、命令详解 |
| [10](./10-开发指南.md) | 开发指南 | 本地开发、调试技巧、扩展开发 |

### 附录与进阶篇

| 章节 | 标题 | 描述 |
|------|------|------|
| [11](./11-常见问题.md) | 常见问题 | FAQ 常见问题解答 |
| [12](./12-参考资源.md) | 参考资源 | 相关文档、链接资源 |
| [13](./13-限流机制与封号避免.md) | 限流机制与封号避免 | API 重试、配额管理、避免限流策略 |
| [14](./14-隐藏功能与宠物系统.md) | 隐藏功能与宠物系统 | Buddy 宠物系统、Auto Dream、Bridge、Ultrareview 等 |

## 🛠 技术栈

- **运行时**: Bun >= 1.2.0
- **语言**: TypeScript / TSX
- **UI 框架**: React / Ink (终端 UI)
- **API**: Anthropic Claude API
- **构建**: esbuild

## 📋 章节预览

### 第一章：项目介绍
- Claude Code 概述
- 版本历史 (1.0 → 2.1.31)
- 反编译项目说明

### 第二章：架构概览
- 整体目录结构
- 核心模块说明
- 技术栈介绍
- 构建方式

### 第三章：入口与启动
- cli.tsx 入口点
- main.tsx CLI 定义
- init.ts 初始化流程
- setup.ts 环境配置

### 第四章：查询引擎
- query.ts 核心查询逻辑
- QueryEngine.ts 查询引擎
- API 客户端协作

### 第五章：工具系统
- Tool.ts 工具接口
- 工具注册机制
- 工具执行模型

### 第六章：UI与REPL
- REPL 屏幕实现
- 消息渲染
- 输入处理
- 权限系统

### 第七章：状态管理与MCP
- Zustand 状态管理
- MCP 协议支持
- 权限管理

### 第八章：工具实现示例
- BashTool 深度解析
- FileEditTool 深度解析
- GrepTool 深度解析

### 第九章：使用指南
- 安装配置
- 交互方式
- 命令详解

### 第十章：开发指南
- 开发环境设置
- 调试技巧
- 创建新工具

### 第十一章：常见问题
- 安装和运行问题
- 使用问题
- 开发问题

### 第十二章：参考资源
- 官方文档
- 技术参考
- 学习资源

### 第十三章：限流机制与封号避免
- API 重试机制
- 配额状态管理
- 避免限流策略

### 第十四章：隐藏功能与宠物系统
- Buddy 宠物��统（18种物种、稀有度系统）
- Auto Dream 记忆整合
- Bridge 远程控制
- Teleport 会话迁移
- Ultrareview 代码审查
- Ultraplan 规划代理
- Kairos 主动式助手

## 🚀 快速开始

```bash
# 克隆文档仓库
git clone https://github.com/fysh1010/claude-code-whitepaper.git

# 查看目录结构
ls -la

# 阅读 README 开始学习
cat README.md
```

## 📚 推荐阅读顺序

### 新手小白
1. **先阅读 架构概览** - 了解项目整体结构
2. **再阅读 入口与启动** - 理解程序如何运行
3. **然后阅读 使用指南** - 学会使用 Claude Code
4. **最后阅读 工具系统** - 理解工具如何工作

### 技术人员
1. **架构概览** - 快速了解结构
2. **入口与启动** - 理解启动机制
3. **查询引擎** - 深入核心逻辑
4. **工具系统** - 理解扩展机制
5. **开发指南** - 开始开发
6. **隐藏功能与宠物系统** - 探索隐藏功能

## 🔧 相关项目

- [claude-code-cli](https://github.com/fysh1010/claude-code-cli) - Claude Code 反编译源码
- [ai-agent-deep-dive](https://github.com/tvytlx/ai-agent-deep-dive) - AI Agent 源码深度研究

## 📄 许可证

本白皮书和项目代码仅供学习参考，不提供任何保证。使用者应自行承担使用风险。

此项目是反编译的代码，用于学习和研究目的。使用前请评估相关法律风险。

---

> 💡 **提示**：建议在学习过程中配合阅读实际源代码，文档中的代码示例都标注了具体文件路径。

## ⭐ 支持

如果这个项目对您有帮助，请 Star 支持一下！

---

*本白皮书基于 Claude Code 2.1.31 版本编写*