---
title: "Hermes Agent笔记：自学习Agent到底学了什么"
date: 2026-04-20T20:00:00+08:00
draft: false
categories: [Agent]
series: [LLM与Agent学习笔记]
tags: [Agent, Hermes Agent, Skill, Memory, 自学习]
summary: "阅读Hermes Agent，重点理解它所谓的自学习闭环：从经验生成skill、改进skill、跨会话记忆和用户模型。"
---

# Hermes Agent笔记：自学习Agent到底学了什么

Hermes Agent 是 Nous Research 做的开源 Agent。它最吸引我的地方不是 MCP，也不是多工具接入，而是它一直强调 self-improving：Agent 会从经验中创建 skills，在使用中改进 skills，持久化知识，还能搜索过去会话。

这个说法很容易被理解成“模型自己训练自己”。但看下来，我觉得更准确的理解是：Hermes 的自学习主要发生在系统层，不是每次都更新底层模型参数。

## 自学习不是模型训练

“Self-improving AI”这个词容易让人想到模型权重自动更新。但大多数个人 Agent 没有条件频繁微调模型，也不应该随便在用户机器上改模型。

Hermes Agent 更实际的路线是：

- 从任务过程中总结经验。
- 把经验写入 memory。
- 把稳定流程整理成 skill。
- 后续任务触发 skill。
- 使用中继续修正 skill。
- 通过历史会话搜索找回上下文。

这是一种外部记忆和技能层的自学习。模型本身可能没变，但 Agent 系统的可用经验变多了，所以表现会更贴合用户。

这个思路很工程化，也更可信。真正长期使用 Agent 时，最需要的往往不是模型“突然变强”，而是它别忘记我怎么做事。

## 学习闭环

Hermes 的学习闭环可以按这样理解：

```text
执行任务 -> 观察结果 -> 总结经验 -> 写入memory或skill -> 后续调用 -> 使用中修正
```

关键是闭环。很多 Agent 也能保存聊天记录，但只是存档，不一定会主动从记录里提炼经验。Hermes 强调的是把经验变成可复用资产。

比如一次部署任务失败了，原因是某个服务启动前必须先设置环境变量。普通聊天系统可能只是这次记住了。自学习 Agent 应该把这条经验沉淀下来，下次遇到同类任务时主动检查。

这才算“学到了”。

## Skills from experience

Hermes 的一个核心点是从经验创建 skills。

Skill 不是普通记忆，而是可复用流程。比如：

- 某个项目如何发布。
- 某类日志如何排查。
- 某个用户喜欢什么格式的总结。
- 某个工具调用前需要哪些准备。

如果一段经验被整理成 skill，Agent 下次就不需要重新探索。

这和人做事很像。第一次做一个流程时，需要边查边试；第二次会参考笔记；第三次就形成 checklist。Hermes 想让 Agent 也有这个过程。

## Skill self-improvement

只创建 skill 还不够。因为 skill 可能不完整，也可能过期。

Hermes 强调 skills 会在使用中改进。也就是说，当某个 skill 被调用后，如果发现步骤漏了、命令变了、用户纠正了某个偏好，就应该更新 skill。

这点很重要。很多自动化流程写完后会慢慢腐烂，因为环境变了。自学习 Agent 如果不能修正旧技能，长期运行反而会积累错误。

好的 skill self-improvement 至少要处理：

- 新增有效步骤。
- 删除过期步骤。
- 标记适用条件。
- 记录失败案例。
- 避免把一次性经验写成通用规则。

最后一点最难。不是每次失败都值得写进 skill。要区分偶发问题和稳定规律。

## 三层记忆

Hermes 文档里把 memory 分成几层来讲，这个结构比较有参考价值。

第一层是始终注入的持久记忆。它适合保存非常稳定的信息，比如用户偏好、固定身份、长期项目背景。

第二层是 skills。它保存可复用任务经验，不一定每次都加载，而是按任务触发。

第三层是 session search。过去会话进入可搜索索引，需要时再查出来。

这三层对应不同时间尺度：

- 长期稳定事实。
- 可复用流程。
- 具体历史记录。

我觉得这个分层比“一个向量库保存全部记忆”更清楚。不同记忆应该有不同加载策略。

## 用户模型

Hermes 还强调会逐渐形成对用户的理解。这个地方很有用，也有风险。

有用的地方在于：Agent 知道用户偏好后，沟通成本会下降。比如喜欢简短回答、偏好某种代码风格、常用某个项目路径、发布前必须先构建。

风险在于：用户模型可能过度推断，也可能保存不该保存的信息。Agent 不应该把所有行为都当成永久偏好。用户这次要简短，不代表永远要简短；某个临时项目路径，也不该变成全局规则。

所以用户模型也需要可解释、可编辑、可删除。长期 Agent 如果不能让用户控制记忆，会变得不可信。

## 和普通Agent的区别

普通 Agent 的重点是完成当前任务。Hermes 更强调跨任务成长。

差别可以这样看：

```text
普通Agent：这次任务做完就结束。
Hermes式Agent：做完后提炼经验，下次用得上。
```

这不是模型参数层面的学习，而是 workflow 层面的学习。对个人 Agent 来说，这可能更实际。

长期使用中，一个 Agent 是否好用，取决于它能不能记住：

- 我是谁。
- 我有哪些项目。
- 我常用什么流程。
- 我不喜欢哪些回答方式。
- 哪些错误之前已经踩过。

这些都不是单次 prompt 能解决的。

## 自学习的边界

自学习也不能无限放开。

我觉得至少有几个边界：

- 写入 memory 前要判断是否稳定。
- 修改 skill 时要保留历史或版本。
- 敏感信息不要自动长期保存。
- 用户纠正应高优先级。
- 重要 workflow 更新最好可审查。
- 不要把模型猜测写成事实。

如果没有边界，自学习会变成自我污染。Agent 越用越偏，最后很难纠正。

所以 Hermes 这类系统最难的不是“自动保存”，而是“保存什么、何时更新、如何撤销”。

## 记录

Hermes Agent 的重点我会记成一句话：它的自学习不是神秘的模型进化，而是把经验沉淀成 memory 和 skills，并在后续任务里重新使用和修正。

这个方向很像个人知识管理加自动化工作流。Agent 真正成长的地方，不是每次回答都更华丽，而是越来越少重复犯错，越来越懂当前用户和当前环境。

## 参考

- [Hermes Agent Documentation](https://hermes-agent.nousresearch.com/docs/)
- [Nous Research Releases: Hermes Agent](https://nousresearch.com/releases/)
- [Hermes Agent GitHub](https://github.com/NousResearch/hermes-agent)
- [How Hermes Agent Memory Works](https://hermes-agent.ai/blog/hermes-agent-memory-system)
- [Self-Improving AI: The Hermes Feature That Actually Works](https://hermes-agent.ai/blog/self-improving-ai-guide)
