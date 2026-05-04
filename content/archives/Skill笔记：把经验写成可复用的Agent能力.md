---
title: "Skill笔记：把经验写成可复用的Agent能力"
date: 2026-02-08T20:00:00+08:00
draft: false
categories: [Agent]
series: [LLM与Agent学习笔记]
tags: [Agent, Skill, Claude, Codex, 工作流]
summary: "整理Skill的基本理解：它和prompt、RAG、MCP的区别，以及为什么适合沉淀可复用任务经验。"
---

# Skill笔记：把经验写成可复用的Agent能力

最近看 Agent 工具时，经常看到 Skill 这个概念。Claude 有 Skills，Codex 里也有技能机制，Hermes Agent 也强调从经验里创建和改进 skills。

我现在的理解：Skill 不是简单 prompt，也不是知识库，而是给 Agent 按需加载的一组任务经验。它可以包括说明、约束、脚本、模板、资源和工作流。

## 为什么需要Skill

如果每次都把所有规则塞进系统 prompt，会有几个问题：

- 上下文变长。
- 不相关规则干扰当前任务。
- 更新经验不方便。
- 很难复用一套流程。

如果只靠 RAG，检索到的是知识片段，不一定能告诉 Agent 怎么完成任务。

Skill 的作用是把一类任务的经验打包起来。比如：

- 怎么写周报。
- 怎么做代码 review。
- 怎么生成 PPT。
- 怎么分析 Excel。
- 怎么按项目规范发布版本。
- 怎么调试某类服务。

当任务触发这个技能时，Agent 加载对应说明和资源，然后按里面的流程执行。

## Skill和Prompt

Prompt 是一次请求里的指令。Skill 更像长期维护的任务包。

Prompt 可以这样写：

```text
帮我把这份文档改得更自然。
```

Skill 则会规定：

- 什么情况下使用。
- 处理步骤是什么。
- 需要遵守哪些风格。
- 有哪些模板可以复用。
- 遇到异常怎么处理。
- 结果如何检查。

所以 Skill 比 prompt 更稳定，也更适合团队或个人长期沉淀。

我觉得最关键的是触发条件。一个好的 Skill 不能只写一堆规则，还要告诉 Agent 什么时候该用、什么时候不该用。否则要么根本用不上，要么什么任务都乱触发。

## Skill和RAG

RAG 主要解决“查资料”。Skill 主要解决“怎么做事”。

比如我问项目里某个接口怎么用，RAG 可以从文档里找接口说明。

但如果我要“按照团队规范写一篇事故复盘”，RAG 找到几篇历史复盘不一定够，还需要知道固定结构、注意事项、检查清单、措辞风格。这更像 Skill。

当然 Skill 里也可以引用资料。比如一个 `weekly-report` skill 可以带模板文件，一个 `slides` skill 可以带生成脚本，一个 `code-review` skill 可以提示去查项目规范。

可以粗略记成：

- RAG：取回相关知识。
- Skill：加载任务流程和经验。

## Skill和MCP

MCP 是工具协议，Skill 是任务经验。

MCP server 可以提供工具：

- 读文件。
- 查 GitHub。
- 运行数据库查询。
- 创建日历事件。

Skill 会告诉 Agent 在某类任务中怎么使用这些工具：

- 先查哪些信息。
- 哪些操作需要确认。
- 出错时怎么回退。
- 最终结果怎么组织。

所以 Skill 和 MCP 是互补关系。MCP 让 Agent 有手，Skill 告诉 Agent 这类活怎么干。

如果只有 MCP，没有 Skill，Agent 可能知道有很多工具，但不知道完成任务的最佳流程。如果只有 Skill，没有工具，Agent 只能停留在文本建议。

## Skill的结构

一个比较实用的 Skill 通常包括：

- 名称和描述。
- 触发条件。
- 使用流程。
- 输入要求。
- 输出格式。
- 禁止事项。
- 可选脚本或模板。
- 验证清单。

在 Codex 的技能机制里，`SKILL.md` 通常是入口。Agent 先读技能说明，只在需要时再加载脚本、模板或参考文件。这种“渐进加载”很重要。

如果一个 Skill 一上来就把所有资料全塞进上下文，反而失去意义。好的 Skill 应该先给最小必要说明，再按任务需要展开。

## 技能的粒度

Skill 不能太大，也不能太碎。

太大的 Skill 会变成一本手册。Agent 读了半天，当前任务只用其中一点。

太碎的 Skill 又会导致触发和选择困难。比如把“写 PR 描述”“列变更点”“写测试说明”拆成三个技能，可能不如合成一个“发布 PR”技能。

我更倾向于按任务闭环拆 Skill。一个 Skill 应该能覆盖一类自然任务，从输入到输出形成完整流程。

比如：

- `github:yeet`：把本地变更发布到 GitHub。
- `documents`：编辑 Word 文档。
- `spreadsheets`：处理表格。
- `humanizer`：把文本改得更像人写。

这些技能的边界都比较清楚。

## 技能沉淀

Skill 最大的价值是沉淀经验。

很多工作不是一次性解决，而是反复出现。每次都靠 prompt 重新描述，会浪费时间，也容易遗漏细节。如果把经验整理成 Skill，后面就能复用。

比如一个项目发布流程，第一次可能需要人一步步提醒：

- 先跑测试。
- 再构建。
- 再检查 git diff。
- 不要提交 `.DS_Store`。
- 最后 commit 和 push。

这些规则沉淀成 Skill 后，Agent 下次遇到类似任务就能主动执行。

这也是 Hermes Agent 自学习里 skills 的意义：把完成任务后的经验转成可复用技能，让系统越用越贴合用户。

## 风险

Skill 也可能带来问题。

第一，过期。项目流程变了，旧 Skill 还在按旧方式执行。

第二，误触发。任务只是类似，但 Skill 被错误加载，结果带偏。

第三，规则冲突。多个 Skill 都适用，但要求不同。

第四，权限扩大。Skill 如果包含危险操作流程，可能让 Agent 更容易执行高风险动作。

所以 Skill 需要版本管理和定期清理。它不是写完就永远正确。

## 记录

Skill 可以先理解成 Agent 的“经验包”。

它和几个概念的区别：

- Prompt：一次性指令。
- RAG：检索知识。
- MCP：连接工具。
- Skill：沉淀任务流程。

真正有价值的 Skill 应该来自反复做过的任务，而不是凭空写一堆规范。用过、踩过坑、修正过，再沉淀下来，这样才会让 Agent 变得更可靠。

## 参考

- [Claude Skills](https://support.claude.com/en/articles/12512176-what-are-skills)
- [Anthropic: Equip Claude with new skills](https://www.anthropic.com/news/skills)
- [Model Context Protocol Documentation](https://modelcontextprotocol.io/)
- [Hermes Agent Documentation](https://hermes-agent.nousresearch.com/docs/)
