---
title: "OpenClaw笔记：个人AI Agent的系统边界"
date: 2026-03-29T20:00:00+08:00
draft: false
categories: [Agent]
series: [LLM与Agent学习笔记]
tags: [Agent, OpenClaw, MCP, Gateway, 安全]
summary: "整理OpenClaw这类个人Agent框架的系统结构：channel、gateway、tools、skills、MCP bridge和权限边界。"
---

# OpenClaw笔记：个人AI Agent的系统边界

这段时间开始注意 OpenClaw。它吸引我的不是单个模型能力，而是它代表的一类个人 AI Agent 形态：Agent 不只是 IDE 里的代码助手，也不只是网页聊天框，而是可以接入聊天渠道、工具、skills、MCP，并长期运行在个人环境里的系统。

这类系统的核心问题不是“能不能回答”，而是“能不能安全地替我做事”。

## 个人Agent的形态

普通 chatbot 的交互很简单：用户发消息，模型回消息。

个人 Agent 更像一个常驻系统：

- 可以从多个 channel 接收消息。
- 可以调用本地或远程工具。
- 可以读写文件和状态。
- 可以接 MCP server。
- 可以保存记忆和技能。
- 可以跨会话继续任务。

这让它更接近一个个人自动化入口。用户不一定打开 IDE 或网页，只要在熟悉的聊天渠道里发一句话，Agent 就能在后台处理。

这很方便，但风险也明显上升。因为它离真实账户、真实文件、真实社交渠道更近。

## Channel

Channel 是用户和 Agent 交互的入口。

可能是：

- CLI。
- Web UI。
- Telegram。
- Discord。
- 微信或其他聊天工具。
- Email。

多 channel 的好处是自然。很多任务本来就是在聊天里产生的，比如“帮我整理这个链接”“帮我查一下这个项目”“提醒我处理这个问题”。

但 channel 也带来身份和权限问题。不同渠道的消息是否都应该有同样权限？群聊里的消息能不能触发工具？转发内容是否可信？这些都需要设计边界。

我觉得个人 Agent 不能默认所有入口都一样可信。CLI 和私聊可能权限高，群聊和公开渠道应该权限低。

## Gateway

Gateway 可以理解成连接外部 channel 和内部 Agent 的中间层。

它负责接收消息、识别来源、做鉴权、转发给 Agent，再把结果发回对应渠道。

Gateway 的价值在于隔离。Agent 不应该直接把每个平台的接入逻辑都揉在一起。不同 channel 的认证、消息格式、回调机制都不同，如果没有 gateway，系统会很乱。

一个清楚的 gateway 层至少要处理：

- 用户身份。
- 会话映射。
- 消息格式转换。
- 文件和附件处理。
- 速率限制。
- 权限分级。

个人 Agent 越常驻，gateway 越重要。它是外部世界进入 Agent 的门。

## Tools

OpenClaw 这类系统的能力来自工具。模型自己不能真正操作外部世界，工具才是执行层。

工具可能包括：

- 文件读写。
- 命令执行。
- 浏览器操作。
- 搜索。
- 任务管理。
- 消息发送。
- MCP 工具桥接。

工具设计需要足够具体。越是泛化的工具，越难控制风险。比如“执行任意 shell 命令”很强，但也很危险。相比之下，“列出目录”“读取指定文件”“运行允许列表里的脚本”更容易限制。

我现在更倾向于把高危工具拆小，并加确认。Agent 能力不是越开放越好，个人环境里尤其要保守。

## Skills

Skills 是经验层。工具提供动作，skills 提供流程。

比如一个个人 Agent 可以有：

- 整理会议记录的 skill。
- 发布博客的 skill。
- 查询项目状态的 skill。
- 处理报错日志的 skill。
- 总结论文的 skill。

OpenClaw 如果能把 skills 和 tools/MCP 结合起来，就不只是“能调用工具”，而是能按稳定流程完成任务。

这也是我关注个人 Agent 的原因：真正有用的不是一次性聊天，而是把重复任务沉淀成可复用流程。

## MCP bridge

MCP bridge 让 Agent 可以接入外部 MCP server。这样 OpenClaw 不需要自己实现所有工具，只要通过 MCP 接已有生态。

这层很关键：

- MCP server 负责具体系统接入。
- OpenClaw 负责把这些能力放进个人 Agent 的任务循环。
- Skills 决定什么时候、怎么用这些工具。

但 MCP bridge 也要限制权限。不能因为某个 MCP server 暴露了危险工具，就直接让聊天消息触发。应该有 allowlist、确认机制和日志。

## 权限分层

个人 Agent 最重要的是权限分层。

我会把操作粗略分成几类：

- 只读：搜索、读取公开资料、查看文件。
- 低风险写入：写草稿、创建本地临时文件。
- 中风险操作：修改项目文件、运行测试、调用内部服务。
- 高风险操作：删除文件、发送消息、提交代码、支付、修改线上配置。

不同风险级别应该有不同确认策略。高风险操作不能让模型自由执行。

尤其是多 channel 场景里，一个来自聊天软件的消息不应该天然拥有本机最高权限。

## 日志和可追溯

Agent 做了什么必须可追溯。

至少应该记录：

- 谁发起任务。
- 从哪个 channel 发起。
- 调用了哪些工具。
- 工具参数是什么。
- 结果是什么。
- 哪些操作经过确认。

没有日志，出了问题很难复盘。个人 Agent 不是只在沙箱里说话，它可能真的改文件、发消息、调用服务，所以可追溯性很重要。

## 记录

OpenClaw 这类系统让我更关注个人 Agent 的边界设计。

它的重点不是某个模型回答多好，而是这些层怎么组合：

```text
channel -> gateway -> agent loop -> skills -> tools / MCP -> external systems
```

每一层都要有边界。channel 要区分身份，gateway 要做鉴权，tools 要限制权限，skills 要沉淀流程，MCP bridge 要控制外部能力。

个人 Agent 真正难的是“常驻”和“可行动”。越常驻，越需要记忆和状态；越可行动，越需要权限和审计。

## 参考

- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw MCP Documentation](https://docs.openclaw.ai/cli/mcp)
- [Model Context Protocol Documentation](https://modelcontextprotocol.io/)
- [HermesClaw bridge discussion](https://github.com/NousResearch/hermes-agent)
