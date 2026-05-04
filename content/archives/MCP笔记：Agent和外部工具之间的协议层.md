---
title: "MCP笔记：Agent和外部工具之间的协议层"
date: 2026-01-11T20:00:00+08:00
draft: false
categories: [Agent]
series: [LLM与Agent学习笔记]
tags: [Agent, MCP, Tool Use, Claude, 协议]
summary: "整理MCP的基本概念：client、server、tools、resources、prompts，以及它为什么适合作为Agent工具协议。"
---

MCP 是 Model Context Protocol。Anthropic 在 2024-11 发布这个协议，2025 年之后很多 Agent 工具开始接它。补 Agent 系统时，MCP 是绕不开的一层。

我的理解：MCP 不是一个 Agent，也不是一个模型能力，而是模型应用和外部工具之间的协议层。它想解决的问题是：不要让每个 Agent 都为每个工具单独写一套接入方式。

## 为什么需要协议

没有 MCP 时，一个 Agent 想接 GitHub、数据库、浏览器、文件系统、内部服务，通常要在应用里写很多定制工具。

这样会有几个问题：

- 工具只能给某个 Agent 用，复用性差。
- 工具描述、参数格式、返回格式不统一。
- 权限边界容易散落在业务代码里。
- 不同模型应用重复造轮子。
- 工具升级后，多个 Agent 都要改。

MCP 的思路是把工具能力封装成 server。Agent 应用作为 client，通过统一协议发现和调用工具。

这有点像 LSP 对编辑器和语言服务的关系。编辑器不需要为每门语言都从头实现补全、跳转和诊断，而是通过协议连接语言服务器。MCP 也类似：Agent 不必直接懂每个外部系统，只要会和 MCP server 通信。

## Client和Server

MCP 里有两个核心角色：

- MCP client：通常嵌在 Claude Desktop、Claude Code、Codex 或其他 Agent 应用里。
- MCP server：负责暴露工具、资源和提示词。

用户使用 Agent 时，client 会连接一个或多个 server。server 告诉 client 自己有哪些能力，client 再把这些能力提供给模型使用。

比如一个 GitHub MCP server 可能暴露：

- 查询 issue。
- 读取 PR。
- 创建 comment。
- 搜索仓库。

一个数据库 MCP server 可能暴露：

- 查看 schema。
- 执行只读 SQL。
- 查询指定表。

关键是 server 可以独立开发、独立部署、独立授权。Agent 应用只要按协议接入。

## Tools

Tools 是 MCP 里最直观的能力。它表示模型可以调用的动作。

一个 tool 通常包括：

- 名称。
- 描述。
- 输入 JSON schema。
- 返回结果。

工具描述很重要。模型并不是直接读代码，它根据工具名称、描述和 schema 决定什么时候调用。如果描述含糊，模型可能用错工具；如果 schema 不清楚，参数就容易错。

我现在会把 tool 看成 Agent 的动作接口。动作接口要足够窄，不能什么都塞进去。比如与其给一个 `run_anything`，不如给几个边界更明确的工具：

- `search_issues`
- `get_pull_request`
- `list_recent_commits`
- `create_review_comment`

接口越具体，权限越容易控制，模型也更不容易乱用。

## Resources

Resources 表示可以读取的上下文资源。它不一定是一个动作，更像可供模型查看的数据。

比如：

- 文件内容。
- 文档页面。
- 数据库 schema。
- 项目配置。
- 运行环境信息。

Tools 是“做什么”，resources 更像“看什么”。这个区分很有价值。很多信息只应该被读取，不应该被修改。如果全部做成 tool，边界会变模糊。

## Prompts

MCP 还支持 prompts。它可以把常用提示词模板作为 server 能力暴露出来。

这点一开始不太显眼，但在团队场景里有用。比如一个内部 MCP server 可以提供：

- 生成 PR 描述的 prompt。
- 事故复盘 prompt。
- SQL 查询分析 prompt。
- 代码 review prompt。

这样 prompt 不必散落在每个客户端里，而是随工具和资源一起被管理。

## 传输方式

MCP 常见传输方式包括 stdio 和 HTTP。

Stdio 适合本地工具。客户端启动 server 进程，通过标准输入输出通信。比如本地文件系统、命令行工具、开发环境集成。

HTTP 适合远程服务。server 部署在远端，client 通过网络访问。比如企业内部服务、云数据库、团队知识库。

这两种方式对应不同风险。stdio 更接近本机权限，HTTP 更涉及网络认证和访问控制。不能只看功能是否可用，还要看运行位置和权限边界。

## 权限边界

MCP 最重要的工程问题其实是权限。

协议让工具接入更方便，也意味着模型更容易触达外部系统。如果 server 暴露过强能力，Agent 就可能执行危险操作。

我觉得至少要注意：

- 默认只读，写操作单独授权。
- destructive 操作需要用户确认。
- server 不要暴露不必要的环境变量。
- 密钥不要进入模型上下文。
- 工具返回结果要脱敏。
- 远程 MCP 要有认证和审计。

MCP 只是协议，不会自动保证安全。安全取决于 server 怎么实现、client 怎么授权、用户怎么配置。

## 和Function Calling的区别

Function calling 是模型 API 层的工具调用机制。开发者在一次请求里定义可调用函数，模型返回函数名和参数。

MCP 更像工具发现和连接协议。它不只描述一次请求里的函数，还定义了 client 如何连接 server、发现 tools/resources/prompts、调用能力。

可以粗略理解：

- Function calling：模型调用函数的格式。
- MCP：Agent 应用连接外部工具生态的协议。

两者不冲突。MCP server 暴露工具，client 最后仍然可能通过模型的 tool calling 能力让模型选择调用哪个工具。

## 和Agent系统的关系

Agent 需要工具，MCP 让工具接入标准化。

如果把 Agent 看成：

```text
模型 + 任务循环 + 工具 + 状态 + 权限
```

那 MCP 主要解决“工具”和一部分“上下文资源”的标准化问题。它不负责完整任务规划，也不负责 memory，但它让 Agent 可以更容易接入外部能力。

这也是为什么 MCP 会和 Claude Code、Cursor、Codex、Hermes Agent、OpenClaw 这些系统一起被讨论。大家都需要同一个问题：怎么让 Agent 可靠地连接真实工具。

## 记录

MCP 的核心价值不是让模型更强，而是让工具生态更容易复用。

我先记住几个概念：

- client 在 Agent 应用里。
- server 暴露 tools、resources、prompts。
- tools 是动作，resources 是上下文，prompts 是模板。
- stdio 适合本地，HTTP 适合远程。
- 协议降低接入成本，但不自动解决安全。

看 Agent 工程时，MCP 是一个很重要的分层点：模型负责推理，Agent loop 负责任务推进，MCP server 负责把外部系统包装成可调用能力。

## 参考

- [Anthropic: Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol)
- [Model Context Protocol Documentation](https://modelcontextprotocol.io/)
- [MCP Specification](https://spec.modelcontextprotocol.io/)
- [Claude Code MCP docs](https://docs.anthropic.com/en/docs/claude-code/mcp)
