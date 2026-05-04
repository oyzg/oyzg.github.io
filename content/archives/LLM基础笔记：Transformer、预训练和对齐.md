---
title: "LLM基础笔记：Transformer、预训练和对齐"
date: 2025-10-12T20:00:00+08:00
draft: false
categories: [LLM]
series: [LLM与Agent学习笔记]
tags: [LLM, Transformer, 预训练, 对齐, RLHF]
summary: "整理LLM的基本训练链路：Transformer、next token prediction、预训练、指令微调和对齐。"
---

这段时间补 LLM 的基础，发现很多概念如果只看名字会很散：Transformer、预训练、指令微调、RLHF、DPO、上下文学习，好像每个都能单独讲很久。但如果从“一个语言模型怎么从文本预测器变成可用助手”这条线去看，结构会清楚很多。

我先按自己的理解记一遍，不追求把所有细节都写全，重点是把主线搭起来。

## Transformer

LLM 的底座基本还是 Transformer。Transformer 论文是 2017 年的 *Attention Is All You Need*，它最核心的变化是用 self-attention 替代传统 RNN 里按时间一步步传递状态的方式。

Self-attention 的直觉是：每个 token 在更新表示时，可以直接看序列里的其他 token，并根据相关性分配权重。比如一句话里某个代词到底指谁，某个动词和哪个主语相关，模型可以通过 attention 在上下文中建立联系。

Transformer 里有几个关键部件：

- Token embedding：把离散 token 映射成向量。
- Positional encoding / position embedding：补充顺序信息。
- Multi-head attention：从多个子空间看 token 之间的关系。
- Feed-forward network：对每个位置的表示做非线性变换。
- Residual connection 和 layer normalization：让深层网络更容易训练。

现在看起来这些结构很普通，但它解决了两个问题：一是并行训练更容易，二是长距离依赖不再完全依靠循环状态传递。后面 GPT、LLaMA、Qwen、DeepSeek 这些模型，本质上都还在这套结构上不断扩展。

## Next token prediction

LLM 预训练的目标很简单：给定前面的 token，预测下一个 token。

形式上就是最大化：

```text
P(x_t | x_1, x_2, ..., x_{t-1})
```

刚开始我会觉得这个目标太弱了。只是猜下一个词，为什么能学会翻译、写代码、推理、总结、规划？后来想明白一点：文本本身压缩了大量世界知识和任务过程。要把下一个 token 预测好，模型不得不学习语法、事实、风格、代码结构、推理模式，甚至人类在文本里留下的解决问题步骤。

所以 next token prediction 表面上是语言建模，实际是在大规模文本上做一种压缩学习。模型不是显式地学一个数据库，也不是显式地学一个规则系统，而是把很多统计规律和模式压进参数里。

这里也要注意边界。模型会学到很多模式，但它不等于可靠知识库。它生成的是概率上合理的续写，不一定是真实答案。这也是后面 RAG、工具调用和 Agent 系统要补的部分。

## 预训练

预训练阶段主要解决“能力来源”的问题。模型在海量文本、代码、网页、书籍、论文等数据上训练，学语言、知识、代码和基本推理模式。

预训练的几个关键词：

- 数据规模：模型能力很大程度来自数据覆盖面。
- 参数规模：更大模型通常有更强表达能力，但训练和推理成本更高。
- 计算规模：训练 token 数、batch、学习率、并行策略都很关键。
- 数据质量：低质量数据会带来噪声、幻觉和偏见。

Scaling law 的意义在于，它让模型训练变得更像工程问题：给定计算预算，怎么分配参数量和训练 token，怎么估计能力提升。Chinchilla 之后，一个重要认识是模型不能只堆参数，还要保证足够训练 token。

预训练出来的 base model 通常不是好用助手。它更像一个强大的文本续写器，能补全，也能模仿，但不一定听指令，不一定按人类偏好回答，也不一定知道什么时候拒绝。

## 指令微调

指令微调，也就是 SFT，解决的是“模型是否听得懂任务格式”的问题。

预训练模型看到的文本形式很多，但它未必自然知道用户说“帮我总结一下”时应该输出什么。SFT 会用人工构造或筛选的 instruction-response 数据继续训练模型，让它学会按照指令回答。

SFT 的效果通常很直观：

- 回答更像助手，而不是续写网页。
- 更容易遵循格式要求。
- 对问答、总结、翻译、代码解释这类任务更稳定。
- 对多轮对话有基本适配。

但是 SFT 也有边界。它学的是示范答案的分布，如果示范数据里没有覆盖复杂偏好，模型不一定知道哪种回答更好。比如同一个问题，答案 A 更有帮助，答案 B 更安全，答案 C 更简洁，SFT 很难直接表达这种偏好比较。

## RLHF

RLHF 解决的是“偏好对齐”的问题。InstructGPT 是这条路线的代表工作。

大致流程是：

1. 先做 SFT，让模型会按指令回答。
2. 对同一个 prompt 采样多个回答，让人类标注哪个更好。
3. 用偏好数据训练 reward model。
4. 用强化学习优化语言模型，让它生成 reward 更高的回答。

RLHF 的目标不是让模型知道更多事实，而是让它更符合人类偏好：更有帮助、更少胡说、更安全、更符合对话习惯。

这里我觉得最关键的点是：RLHF 把“好回答”从固定标签变成了偏好比较。很多自然语言任务没有唯一标准答案，用偏好学习会更自然。

但 RLHF 也复杂。它需要 reward model，需要 PPO 这类训练过程，工程成本高，也可能出现 reward hacking。模型可能学会迎合 reward，而不是真正变可靠。

## DPO

DPO 是 2023 年的工作，题目是 *Direct Preference Optimization*。它的思路是绕开显式 reward model 和 RL 过程，直接用偏好数据优化模型。

DPO 的直觉是：如果对于同一个 prompt，人类更喜欢回答 y_w 而不是 y_l，那么模型应该提高 y_w 的概率，降低 y_l 的概率，同时不要偏离参考模型太多。

相比 RLHF，DPO 的工程链路简单很多：

- 不需要单独训练 reward model。
- 不需要跑复杂强化学习。
- 更像常规监督训练，稳定性更好。

所以现在很多模型对齐流程会使用 DPO 或类似偏好优化方法。它不是完全替代 RLHF 的所有情况，但让偏好对齐更容易工程化。

## 上下文学习

In-context learning 是 LLM 很特别的能力。不给模型更新参数，只是在 prompt 里放几个例子，模型就能模仿任务形式完成新任务。

这个能力来自预训练。训练时模型见过大量上下文模式，所以推理时能把 prompt 当作临时任务描述。

比如：

```text
输入：苹果
输出：水果

输入：篮球
输出：运动器材

输入：钢琴
输出：
```

模型可以根据前两个例子推断第三个应该输出“乐器”。这不是参数更新，而是上下文里的模式归纳。

上下文学习的价值很大，因为很多任务不需要微调，只要 prompt 设计得好就能跑。但它也有不稳定的问题：例子顺序、表述方式、上下文长度都会影响结果。

## 能力和幻觉

LLM 的强大和不可靠来自同一个机制：它根据上下文生成高概率 token。

当训练数据覆盖充分、任务模式清晰时，这个机制表现得像理解和推理。当问题需要精确事实、实时信息、外部状态或严格执行时，它就容易出问题。

所以我不想把 LLM 单独看成完整系统。Base model、SFT、RLHF/DPO 只是把模型变成一个更会回答问题的核心组件。真正落到应用里，还需要：

- RAG 提供外部知识。
- 工具调用处理实时状态和计算。
- Agent loop 处理多步任务。
- Memory 保存跨会话经验。
- 权限系统限制危险操作。

LLM 是核心，但不是全部。

## 记录

LLM 的主线可以先记成四层：

第一层是 Transformer，提供可扩展的序列建模结构。

第二层是预训练，用 next token prediction 从大规模文本中学通用模式。

第三层是指令微调，让模型从续写器变成能听指令的助手。

第四层是对齐，用 RLHF、DPO 等方法让回答更符合人类偏好。

后面看 RAG、Agent、MCP、Skill 时，其实都是在补 LLM 的系统边界。模型本身负责语言和推理，外部系统负责知识、工具、记忆、权限和执行。

## 参考

- [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
- [Language Models are Few-Shot Learners](https://arxiv.org/abs/2005.14165)
- [Training language models to follow instructions with human feedback](https://arxiv.org/abs/2203.02155)
- [Direct Preference Optimization](https://arxiv.org/abs/2305.18290)
- [Training Compute-Optimal Large Language Models](https://arxiv.org/abs/2203.15556)
