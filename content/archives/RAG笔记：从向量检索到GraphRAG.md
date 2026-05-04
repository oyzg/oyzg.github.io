---
title: "RAG笔记：从向量检索到GraphRAG"
date: 2025-11-09T20:00:00+08:00
draft: false
categories: [LLM]
series: [LLM与Agent学习笔记]
tags: [LLM, RAG, GraphRAG, 向量检索, 知识库]
summary: "整理RAG的工程流程：切分、向量化、召回、重排、上下文拼接、生成和GraphRAG。"
---

RAG 的全称是 Retrieval-Augmented Generation。原始 RAG 论文是 2020 年的 *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*。这两年实际做知识库问答时，RAG 已经不只是一个论文方法，更像 LLM 应用里的基础工程模块。

我的理解：RAG 不是为了让模型“更聪明”，而是为了让模型在回答时能拿到外部资料。模型负责理解和组织语言，检索系统负责把相关资料找出来。

## 为什么需要RAG

LLM 预训练参数里有很多知识，但这些知识有几个问题：

- 不够新：训练截止后的信息不在参数里。
- 不够准：模型可能记错，也可能编造。
- 不够私有：公司文档、个人笔记、项目代码不在公开训练数据里。
- 不可追溯：直接生成的答案很难知道依据来自哪里。

RAG 的思路是：不要指望模型把所有知识都记在参数里，而是在回答前先查资料，把查到的内容作为上下文交给模型。

基本流程：

```text
用户问题 -> 查询改写 -> 检索相关文档 -> 重排 -> 拼接上下文 -> LLM生成回答
```

这样模型回答时至少有材料可依赖。如果答案需要引用某个项目文档、接口说明或论文段落，RAG 比纯模型生成更稳。

## 文档切分

RAG 工程里第一步通常是文档切分。原始文档可能很长，不能直接全部塞进上下文，所以要切成 chunks。

切分看起来简单，但实际很影响效果。

切太大，召回粒度粗，一个 chunk 里可能混了很多主题，模型拿到后不容易聚焦。

切太小，上下文不完整，召回到的片段可能缺少前后解释。

常见切分方式：

- 固定长度切分：实现简单，但容易切断语义。
- 按标题切分：适合 Markdown、文档站、论文。
- 按段落切分：语义比较自然。
- 滑动窗口切分：通过 overlap 保留上下文连续性。

我更倾向于先按文档结构切，再用 token 长度做兜底。比如 Markdown 就按标题层级切，代码文档按函数或类切，FAQ 按问答对切。

## Embedding和向量库

切完 chunk 后，需要用 embedding 模型把文本转成向量。向量的目标是让语义相近的内容在向量空间里距离更近。

用户查询也会转成向量，然后在向量库里找相似 chunk。

这里有几个容易踩坑的点：

- embedding 模型要和语言匹配。中文、英文、代码、多语言场景差异很大。
- 文档和 query 的写法可能不一样。用户问的是问题，文档写的是陈述句。
- 只靠向量相似度可能召回不到关键词很强但语义表达不同的内容。
- chunk 里 metadata 很重要，比如标题、路径、更新时间、权限。

向量库只是索引，不是完整答案系统。很多 RAG 效果差，不是因为向量库不行，而是 chunk、embedding、query、metadata 和 rerank 没做好。

## Hybrid search

纯向量检索有时会漏掉精确词。比如查某个函数名、错误码、配置项，关键词匹配反而更可靠。

所以很多工程系统会做 hybrid search：

- 向量检索：适合语义相似。
- BM25 / keyword search：适合精确词。
- metadata filter：适合按项目、时间、权限过滤。

然后把不同召回结果合并，再进入 rerank。

我现在比较认可 hybrid search。因为真实问题不全是自然语言问答，很多问题带有代码名、日志、报错、接口路径，这些信息不能只靠语义向量。

## Rerank

召回阶段追求“不要漏”，rerank 阶段追求“排得准”。

向量库可能先召回 top 50 或 top 100，再用 reranker 判断每个 chunk 和 query 的相关性，选出 top 5 或 top 10 交给 LLM。

Reranker 可以是 cross-encoder，也可以是 LLM 自己做排序。它通常比向量相似度更准，因为它能同时看 query 和 chunk 的完整文本。

RAG 里经常出现一种问题：召回结果里确实有正确材料，但排在后面，最终没进入上下文。这种情况下，提升 rerank 往往比换大模型更有效。

## 上下文拼接

把检索结果交给 LLM 前，还要组织上下文。

常见做法是：

- 保留来源标题和路径。
- 按相关性排序。
- 控制总 token 数。
- 对重复内容去重。
- 在 prompt 中要求基于资料回答。

上下文不是越多越好。塞太多材料会稀释注意力，也可能引入互相矛盾的信息。尤其是长文档问答，模型容易抓住不相关段落生成答案。

我更喜欢把 RAG prompt 写得明确一点：

```text
根据下面资料回答。如果资料不足，就说明不足。
不要编造资料中没有的事实。
```

这不能完全消除幻觉，但能减少模型脱离材料自由发挥。

## 评估

RAG 不能只靠主观感觉。比较重要的评估维度：

- Retrieval recall：正确材料有没有被召回。
- Context precision：上下文里无关内容多不多。
- Answer correctness：答案是否正确。
- Faithfulness：答案是否忠于检索材料。
- Citation accuracy：引用来源是否对应。

如果答案错，要拆开看是检索错、排序错、上下文组织错，还是模型生成错。

这一点很关键。RAG 是一个链路，最后答案不好不一定是模型问题。很多时候是文档没切好，或者正确 chunk 没进上下文。

## GraphRAG

GraphRAG 是 2024 年之后比较常见的思路。它不是只把文档切成独立 chunk，而是从文本中抽取实体和关系，构建知识图谱，再用图结构帮助检索和总结。

普通向量 RAG 更擅长回答局部问题，比如“某个配置怎么写”。GraphRAG 更适合跨文档、跨实体的全局问题，比如“这个项目里有哪些核心模块，它们如何依赖”“某个主题下不同文档的观点关系是什么”。

GraphRAG 大致会做几件事：

- 从文档中抽取实体。
- 抽取实体之间的关系。
- 构建社区或主题簇。
- 对社区生成摘要。
- 查询时结合图结构和文本片段回答。

它的优势是能把散落在文档里的信息组织起来。缺点也明显：构图成本高，抽取会有错误，更新也更复杂。

所以我不会把 GraphRAG 当作普通 RAG 的完全替代。它更适合知识密集、关系复杂、需要全局总结的场景。

## RAG和Agent

普通 RAG 通常是一次检索一次回答。Agentic RAG 会更进一步，让模型自己决定是否继续查、换关键词查、打开哪个工具、比较多个来源。

这种方式更像搜索过程：

```text
提出问题 -> 搜索 -> 阅读 -> 判断是否足够 -> 继续搜索 -> 汇总回答
```

优点是复杂问题上更灵活。缺点是成本更高，延迟更长，也更容易跑偏。所以 Agentic RAG 需要更强的状态管理和停止条件。

## 记录

RAG 的核心不只是“向量库 + LLM”。真正要做稳，需要把切分、召回、重排、上下文拼接、生成约束和评估串起来。

我目前会这样理解：

- 向量检索解决语义召回。
- 关键词检索补精确匹配。
- Rerank 决定哪些材料真正进入上下文。
- Prompt 约束模型基于资料回答。
- GraphRAG 补结构化和全局关系。

RAG 本质上是在给 LLM 补外部知识边界。模型不用记住全部事实，但回答时必须能查到、读到、引用到。

## 参考

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Dense Passage Retrieval for Open-Domain Question Answering](https://arxiv.org/abs/2004.04906)
- [REALM: Retrieval-Augmented Language Model Pre-Training](https://arxiv.org/abs/2002.08909)
- [From Local to Global: A Graph RAG Approach to Query-Focused Summarization](https://arxiv.org/abs/2404.16130)
- [Microsoft GraphRAG](https://github.com/microsoft/graphrag)
