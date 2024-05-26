# RAG

## Motivation

传统生成式模型可以根据其训练语料来生成连续的文本，但在处理需要**大量特定背景信息**的任务时，训练时的数据集就力不从心了。

因此，我们需要一种方法，它可以使大模型**方便且有效地结合外部信息**，进而产出更高质量的回答，这就是 RAG，也有人把它叫做「外挂知识库」。

## Principle

RAG，Retrieval-augmented generation，检索增强生成。

RAG 基于两个核心组件：检索器（Retriever）和生成器（Generator）。检索器负责从一个大型的文档数据库中检索出与输入查询最相关的文档或信息片段，而生成器（实际上就是大模型）则基于这些检索到的文档生成最终的回答或内容。

给定一个查询（Prompt）$q$，检索器首先从数据库 $D$ 中找到 $k$ 个最相关的文档 $\{d_1, d_2, ..., d_k\}$。然后，生成模型将查询 $q$ 和检索到的文档 $\{d_1, d_2, ..., d_k\}$ 作为输入，生成回答 $a$，这个过程即：

$$
P(a|q) = \sum_{i=1}^{k} P(a|q, d_i)P(d_i|q) \tag{1}
$$

其中，$P(d_i|q)$ 表示文档 $d_i$ 在给定查询 $q$ 时的相关性概率，而 $P(a|q, d_i)$ 则是给定查询和文档 $d_i$ 时生成回答 $a$ 的概率。

接下来我们补点数学课。为什么 $(1)$ 用概率形式来表示 RAG 的过程呢？

先尝试理解 $(1)$：计算给定一个查询 $q$ 时，生成回答 $a$ 的概率，而这个概率 $P(a|q)$ 通过两个主要部分组合而成：

- **$P(d_i|q)$**：给定查询 $q$ 时，文档 $d_i$ 被检索出来的概率。检索器的任务是从大量的文档中找到与查询最相关的文档，而 $P(d_i|q)$ 则反映了每个文档与查询的相关性。

实际实现中，可选用不同的检索模型来获取 $P(d_i|q)$，比如基于向量的相似性计算（如使用 BERT embeddings 进行余弦相似度计算）。

- **$P(a|q, d_i)$**：给定查询 $q$ 和检索到的文档 $d_i$ 的情况下，生成回答 $a$ 的概率。生成器（如GPT）会考虑查询和检索到的文档的信息，来生成一个合适的回答。

这个部分其实就反映了大模型利用检索到的文档信息来提高生成回答质量的过程。

最后，整个表达式再对一共 $k$ 个被检索到的文档的贡献进行加和，来计算最终的回答概率。加权和的形式允许生成器在生成回答时考虑到多个检索到的文档，而每个文档的贡献则由其与查询的相关性决定，整个过程也可理解为对模型决策进行建模。

Meta（当时还叫 Facebook）在 RAG 的原始论文 [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks
](https://arxiv.org/abs/2005.11401) 中对 RAG 的原理给出了更详细的介绍。

## Implementation

一个 RAG 实现示例：

```python linenums="1"
import torch
from transformers import RagTokenizer, RagRetriever, RagTokenForGeneration

# 初始化 tokenizer 和生成器
tokenizer = RagTokenizer.from_pretrained("facebook/rag-token-nq")
model = RagTokenForGeneration.from_pretrained("facebook/rag-token-nq", retriever=RagRetriever.from_pretrained("facebook/rag-token-nq", use_dummy_dataset=True))

input_text = "What is the capital of France?"
input_ids = tokenizer(input_text, return_tensors="pt").input_ids

outputs = model.generate(input_ids)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```
其中，tokenizer 用于将文本转换为模型可以理解的格式，即 tokens；RagTokenForGeneration 是 RAG 模型的生成部分，RagRetriever 是 RAG 模型的检索部分。

上述论文 [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks
](https://arxiv.org/abs/2005.11401) 也给出了对应的示例代码，开源于：[GitHub 链接](https://github.com/huggingface/transformers/tree/main/examples/research_projects/rag)。

实际应用中，也有很多开源的 RAG 框架可供快速部署，例如 [RAGFlow](https://github.com/infiniflow/ragflow)、[MaxKB](https://github.com/1Panel-dev/MaxKB) 等。具体地，这些框架提供了从解析器到生成器的一整套解决方案，可以对不同文件格式的外部材料进行本地解析和 tokenize，亦可选择不同相似度的计算方法和后端大模型 API。

## Challenges

RAG 流程中，不同阶段分别有不同的潜在挑战。

对外部材料来说，一个是材料本身是否可靠，集中于质量问题和时效问题；一个是 tokenize 的过程能否将外部材料信息高质量地转换为大模型可利用的 token。

前者实际上更依赖于材料的提供者，而后者的影响因素众多，甚至涉及到多模态问题。例如，外部材料存在不同的文件格式或排版格式，使用的 OCR 等方式能否将其高质量地转换为纯文本信息？实际上，[Implementation](#implementation) 小节中提到的开源 RAG 框架 RAGFlow 已经在不同文件及排版格式上做了一些工作，RAGFlow 的文件解析器 DeepDoc 见 [GitHub 链接](https://github.com/infiniflow/ragflow/tree/main/deepdoc)。

对检索器来说，核心则是检索的速度和准确率。

在市面上常见的 RAG 框架中，一般是使用向量化的方式来计算向量之间余弦距离，或者通过关键词匹配的方式，再或者二者结合均返回结果，选其相关性较高的一个。显然，速度和准确难以得兼，更多是一个结合应用场景再做 trade off 的问题。学界对这个问题的进展我没有做深入调研，我猜测是在检索算法或者索引结构上做一些优化。

对生成器来说，实际上更依赖于大模型的表现，依赖于 Prompt Engineering 或者 Fine Tuning 等方法对模型本身做定向适配。

除此之外，我在实际工作中也遇到了一些**看似和 RAG 强相关但其实已经超出了 RAG 适用范围**的问题，解决思路是用多个 Agents 做配合再结合一些传统算法。具体细节按下不表。

## More

关于 RAG 的综述文章，依照最近更新时间排序。

1. [Submitted on 29 Feb 2024 (v1), last revised 2 May 2024 (this version, v4)]
[Retrieval-Augmented Generation for AI-Generated Content: A Survey](https://arxiv.org/abs/2402.19473)

2. [Submitted on 18 Dec 2023 (v1), last revised 27 Mar 2024 (this version, v5)] 
[Retrieval-Augmented Generation for Large Language Models: A Survey](https://arxiv.org/abs/2312.10997)
