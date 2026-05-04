---
title: "Claude Code笔记：源码泄漏后再看代码Agent"
date: 2026-05-02T20:00:00+08:00
draft: false
categories: [Agent]
series: [LLM与Agent学习笔记]
tags: [Agent, Claude Code, Coding Agent, 源码, 工程安全]
summary: "结合Claude Code源码泄漏事件，重新理解代码Agent的外壳系统：工具组织、权限控制、编辑执行循环和上下文管理。"
---

Claude Code 不是第一个代码 Agent，但它很典型。它在终端里读仓库、改文件、跑命令、根据测试反馈继续修改。2025 年它从 research preview 到 GA，已经说明代码 Agent 从 demo 进入了更实用的工具形态。

源码泄漏事件之后，我更想看的不是“它会不会写代码”，而是代码 Agent 的外壳系统到底怎么组织。因为真正能长期工作的 coding agent，不只是模型强，还要有一整套 harness。

## 事件本身

2026-03-31 前后，Claude Code 的 npm 包因为 source map 问题暴露了可读源码，随后很多分析开始围绕它的系统提示词、工具定义、权限控制和内部组织展开。

这类事件对产品当然是安全事故，但对学习 Agent 架构来说，也让大家更直观看到一个成熟代码 Agent 需要哪些工程部件。

我不想把重点放在猎奇源码细节上。更有价值的是：从这件事反推代码 Agent 的系统设计。

## Coding Agent不是聊天框

代码 Agent 和普通聊天助手的区别很大。

普通聊天只要回答文本。代码 Agent 要在真实仓库里行动：

- 搜索文件。
- 阅读代码。
- 制定修改计划。
- 编辑文件。
- 运行测试。
- 分析失败。
- 再次修改。
- 汇总结果。

这要求系统提供文件工具、shell 工具、diff 管理、上下文压缩、权限确认和任务状态。

所以 Claude Code 的重点不是“Claude 会写代码”，而是把 Claude 放进一个能安全操作项目的终端环境里。

## Agent harness

Harness 可以理解成包在模型外面的执行框架。

它负责：

- 给模型提供系统指令。
- 暴露可调用工具。
- 管理上下文。
- 控制权限。
- 展示 diff。
- 捕获命令输出。
- 处理用户中断。
- 在任务结束时汇总。

模型只负责根据上下文做决策。真正的读写、运行、审批都由 harness 执行。

源码泄漏后最值得关注的就是这层。因为 coding agent 的产品差异，很大一部分就在 harness，而不是模型 API 调用本身。

## 工具组织

代码 Agent 的工具不能太随意。

常见工具包括：

- 文件搜索。
- 文件读取。
- 文件编辑。
- shell 命令。
- 计划更新。
- 网络访问。
- 浏览器或文档查询。

每个工具都应该有明确边界。比如读取文件和编辑文件要分开，普通命令和危险命令要分开，网络访问要受控。

工具返回结果也要整理。测试失败时，Agent 不需要看十万行日志，而需要失败用例、错误栈和关键上下文。搜索文件时，Agent 需要路径和匹配行，不需要无关噪声。

工具设计越清楚，模型越容易做出正确下一步。

## 编辑循环

Claude Code 这类工具最核心的是编辑循环：

```text
理解任务 -> 搜索代码 -> 阅读上下文 -> 修改文件 -> 运行验证 -> 根据反馈继续改
```

这里有几个细节很重要。

第一，不能没读代码就改。代码 Agent 如果直接凭经验生成，会很容易破坏项目风格。

第二，修改要小步。一次改太多，测试失败后很难定位。

第三，要尊重用户已有改动。不能为了任务方便把不相关改动回滚。

第四，要验证。至少跑格式化、构建或相关测试。不能只说“应该可以”。

这其实和一个靠谱工程师的工作方式很像。

## 权限控制

代码 Agent 能运行命令，所以权限控制很关键。

危险点包括：

- 删除文件。
- 修改 git 历史。
- 访问网络。
- 读取密钥。
- 执行安装脚本。
- 推送代码。
- 调用外部服务。

Claude Code 这类工具通常会通过 sandbox、审批、命令 allowlist、工作目录限制等方式控制风险。

源码事件让我更明确一点：Agent 产品的安全边界必须写在系统设计里，不能只靠模型“自觉”。模型会犯错，权限层必须硬。

## 上下文管理

代码仓库很大，不可能全部塞进上下文。Coding agent 必须会选择性读取。

常见策略：

- 先用搜索定位文件。
- 只读相关片段。
- 保留任务摘要。
- 把长日志压缩。
- 在上下文不足时重新读取文件。
- 对长任务做 compact。

上下文管理不只是省 token。它决定模型看到什么，也决定它会忽略什么。读错文件、漏读约束、忘记前面修改，都会导致错误。

所以一个成熟 coding agent 需要持续维护任务状态，而不是每轮都从零开始。

## 系统提示词

源码相关讨论里，系统提示词通常很受关注。但我觉得 prompt 只是其中一部分。

系统提示词会规定：

- 工作方式。
- 文件编辑规则。
- 测试要求。
- 与用户沟通方式。
- 安全限制。
- 遇到不确定时怎么处理。

这些规则很重要，但如果没有工具和权限系统配合，prompt 本身不够。比如提示词说“不要运行危险命令”，但工具层允许任意命令，风险仍然存在。

所以我更愿意把系统提示词看成策略层，把工具和沙箱看成执行边界。

## 源码泄漏的启发

这次事件给我的启发主要有三点。

第一，代码 Agent 的核心资产不只是模型，还有 agent harness、工具设计、prompt、权限策略和上下文管理。

第二，AI 工具自身也是软件产品，同样会有供应链、打包、source map、发布流程等工程安全问题。

第三，越强的 Agent 越需要透明和可审计。用户把仓库和命令行交给它，必须知道它能做什么、做了什么、什么时候需要确认。

## 记录

Claude Code 可以看成 coding agent 的一个成熟样本。它的价值不是简单替我写几行代码，而是把模型接进真实开发循环：

```text
读仓库 -> 改文件 -> 跑验证 -> 看反馈 -> 再修改
```

源码泄漏事件之后，我更关注的是这套系统外壳。模型能力会继续提升，但 coding agent 能不能可靠工作，取决于工具组织、权限边界、上下文管理和验证习惯。

## 参考

- [Claude Code Overview](https://docs.anthropic.com/en/docs/claude-code/overview)
- [Claude Code Release Notes](https://docs.anthropic.com/en/release-notes/claude-code)
- [Claude Code GitHub Action](https://docs.anthropic.com/en/docs/claude-code/github-actions)
- [Model Context Protocol in Claude Code](https://docs.anthropic.com/en/docs/claude-code/mcp)
- [VentureBeat: Claude Code's source code appears to have leaked](https://venturebeat.com/technology/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know/)
