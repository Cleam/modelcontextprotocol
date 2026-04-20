# 📘 MCP（Model Context Protocol）完全学习指南

> **Model Context Protocol**（模型上下文协议）是一个开放标准，用于将 AI 应用与外部数据源和工具无缝集成。  
> 本教程基于 MCP 官方仓库内容编写，带你从零掌握 MCP 的原理、设计与实战。

---

## 🎯 谁适合阅读？

- 希望了解 MCP 协议设计思想的**技术决策者**
- 需要开发 MCP Server / Client 的**后端和全栈开发者**
- 想要为 AI 应用接入外部能力的**AI 应用开发者**
- 对协议设计和分布式系统感兴趣的**技术爱好者**

---

## 📚 目录结构

本教程按照**由浅入深**的顺序组织，建议按序阅读：

| 序号 | 文档                                                          | 内容概要                         |
| ---- | ------------------------------------------------------------- | -------------------------------- |
| 01   | [MCP 概述与入门](./01-introduction.md)                        | 什么是 MCP、解决什么问题、生态   |
| 02   | [架构设计与原理](./02-architecture.md)                        | Host-Client-Server 三层架构      |
| 03   | [协议生命周期](./03-protocol-lifecycle.md)                    | 初始化、能力协商、会话管理       |
| 04   | [消息格式与 JSON-RPC](./04-message-format.md)                 | 消息类型、元数据、错误处理       |
| 05   | [传输机制详解](./05-transport.md)                             | stdio / Streamable HTTP / SSE    |
| 06   | [服务端能力 — 工具（Tools）](./06-server-tools.md)            | 工具定义、调用、安全             |
| 07   | [服务端能力 — 资源（Resources）](./07-server-resources.md)    | 资源发现、读取、订阅             |
| 08   | [服务端能力 — 提示词模板（Prompts）](./08-server-prompts.md)  | 模板定义、参数、使用场景         |
| 09   | [客户端能力](./09-client-features.md)                         | Sampling / Roots / Elicitation   |
| 10   | [安全机制与授权](./10-security.md)                            | OAuth 2.1、安全攻防、最佳实践    |
| 11   | [扩展机制](./11-extensions.md)                                | 扩展标识、协商、生命周期         |
| 12   | [实战：构建 MCP Server](./12-practice-server.md)              | 从零到一构建天气查询服务         |
| 13   | [实战：构建 MCP Client](./13-practice-client.md)              | 实现一个完整的 MCP 客户端        |
| 14   | [进阶主题](./14-advanced-topics.md)                           | Tasks / 版本管理 / Schema / SEP  |

---

## 🧭 阅读路线推荐

```
初学者路线：01 → 02 → 03 → 06 → 12（快速上手）

完整学习路线：01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11 → 12 → 13 → 14

实战速通路线：01 → 02 → 06 → 12 → 13（直接动手）
```

---

## 🔗 参考资源

| 资源           | 链接                                                      |
| -------------- | --------------------------------------------------------- |
| MCP 官方文档   | [modelcontextprotocol.io](https://modelcontextprotocol.io)|
| MCP 规范仓库   | [GitHub](https://github.com/modelcontextprotocol/modelcontextprotocol) |
| TypeScript SDK | [GitHub](https://github.com/modelcontextprotocol/typescript-sdk) |
| Python SDK     | [GitHub](https://github.com/modelcontextprotocol/python-sdk) |
| MCP Inspector  | [GitHub](https://github.com/modelcontextprotocol/inspector) |

---

## 📝 说明

- 本教程基于 MCP 规范版本 **2025-11-25**（当前版本）编写
- 所有代码示例仅用于教学，实际项目请参考官方 SDK
- 专业术语保留英文原文并附中文解释，如 Tool（工具）、Resource（资源）
- 文中使用 Mermaid 图表辅助理解，请使用支持 Mermaid 的 Markdown 阅读器

---

> 💡 **提示**：MCP 的设计深受 LSP（Language Server Protocol）启发。如果你了解 LSP 的工作方式，很多概念会感到似曾相识。
