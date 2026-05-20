---
layout: post
title:  "DeepSeek-V4，让百万上下文进入普惠时代"
date:   2026-04-27 21:52:31 +0800
categories: LLM
---

**DeepSeek V4 终于千呼万唤始出来。** 
与DeepSeek V4 模型同步公布的文章开篇有这么一段话

> This breakthrough enables efficient support for a context length of one million tokens, aiming at next-generation large language models that break the efficiency barrier of ultra-long-context processing. **ushering in a new era of million-length contexts for next-generation LLMs.**
> 
> We believe our capability to efficiently handle ultra-long sequences unlocks the next frontier of test-time scaling, paves the way for deeper research into long-horizon tasks, and **establishes a necessary foundation for exploring future paradigms like online learning.**

如同DeepSeek公众号文章的Slogan，DeepSeek V4团队相信DeepSeek开创了LLM百万上下文的新篇章，为后续诸如在线学习的模型新范式打下了基础。纵观文档，DeepSeek V4整套架构的设计，都是在为加强模型在长下文场景下的能力而考量。

我们先看看文档中给出的DeepSeek V4能力评价：
- **世界知识**：超越主流开源模型，略微落后于 Gemini-3.1-Pro。
- **推理能力**：表现优于 GPT-5.2 和 Gemini-3.0-Pro，但与 GPT-5.4 及 Gemini-3.1-Pro 相比仍有 3～6 个月的差距。
- **Agent 能力**：与顶尖开源模型 Kimi-k2.6、GLM-5.1 齐头并进，接近 Opus-4.5 的水平，属国内第一梯队，仅略逊于闭源顶尖模型 Opus-4.7 和 Gemini-3.1-Pro。
- **长上下文能力**：在学术基准测试和真实场景应用中的表现均十分出色，甚至在某些学术基准上超越了 Gemini-3.1-Pro。

那么，DeepSeek做了哪些优化，来加强模型在长上下文处理方面的能力呢 ? 

# 架构优化

首先，V4 继承了 V3 系列的 **DeepSeekMoE 架构** 与 **MTP (Multi-Token Prediction) 策略**。为了在长上下文窗口下依然保持高效，DeepSeek-V4 采用了**混合注意力机制**、**流形约束超连接 (mHC)** 分别解决长上下文注意力计算复杂度、Hyper Connection带来的训练不稳定问题，采用Muon优化器获得了更高的训练效率与稳定性。

>- **压缩稀疏注意力 (Compressed Sparse Attention)** 和 **重度压缩注意力 (Heavily Compressed Attention)**。相比于 DeepSeek-V3.2，单 token 算力仅为原来的 **27%**，KV 缓存仅占 **10%**，大幅降低了长序列推理成本。
>- **mHC (Manifold-Constrained Hyper-Connections)** 增强了跨层信号传播的稳定性，保证 1.6T 参数超大规模训练的平稳性。
>- **Muon 优化器** 通过矩阵级梯度正交化，进一步加速收敛并提升训练稳定性，尤其适合长序列训练。

![deepseekv4arch](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/deepseekv4arch.png)


我们重点看一下支撑长上下文最重要的混合**注意力机制**
# CSA 和 HCA

首先具象化1M上下文，百万上下文大约相当于75万个英文单词，相当于6~7本哈利波特。针对中文分词做了优化的国产大模型，1M上下文大约相当于120~150万汉字，几乎可以一次读完西游记+红楼梦；

你一定知道LLM的多头注意力计算时，每一个输入的token，都需要计算他与之前的所有token的注意力分数，才能捕捉到这个token本身的含义。在百万上下文这个规模下，计算量将会是惊人的1万亿 ($10^{12}$ )，这样计算不仅没有必要，更大的瓶颈是显存会爆炸。

Transformer 推理时，每个token会把它的 Key 和 Value 向量缓存下来，后续的TOKEN计算注意力时会直接读取缓存计算。序列有 n 个 token，就要存 n 对 K、V，长度为 n 的上下文 → KV cache 有 n 条。上下文越长 → KV 越大，显存爆、带宽也爆。在DeepSeek V4之前的方案是通过压缩K,V向量本身来减少显存使用，但是上下文变长之后，依然对显存消耗巨大。

DeepSeek V4的解法是，把n也进行压缩，CSA的机制是将N个TOKEN的KV缓存，按照每4个一组进行压缩，而后进行Top-K筛选（DeepSeek稀疏注意力机制），而后进行MQA计算。

![CSA](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/CSA.png)

而HCA更狠，按照128个TOKEN为一组压缩，因为KV本身已经被极大的压缩，这部分是压缩后的KV全都进行MQA注意力计算。

![HCA](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/HCA.png)
同时，CSA和HCA的机制都保留了临近的128个TOKEN滑窗，以保证相邻的前序Token的注意力计算是没有任何失真的。

即，
  - 滑动窗口注意力：覆盖最近的 128 个 token
  - CSA：覆盖中等距离的 token
  - HCA：覆盖超长距离的 token

经过这一系列操作，注意力的计算量和显存的消耗极大的减少。


> 💡 **比喻：图书馆的记忆系统**
> 
> 想象一个图书管理员要记住 100 万本书的内容。如果每本书都详细记住（全注意力），大脑会爆炸。CSA 的做法是把每 m 本书压缩成一个"摘要卡片"，只保留关键信息；HCA 更进一步，把大量书籍压缩成极少数"核心笔记"。另外，最重要的128本书，始终摆在管理员面前。这样，管理员既能记住整体脉络，又能精准定位细节。


# mHC (manifold- constraint Hyper Connection)

你一定也知道，为了保证在深层次网络自回归训练时输入层总能学习到梯度，在LLM的架构中，经过Attention和FFN的计算时，原始的输入信息都会直接添加到经过Attention和FFN到输出上，这就是残差在Transformer架构上的应用。

```
y = x + F(x)
```


残差连接主要解决的是梯度消失的问题，
想象一群人排成一列玩传话游戏。信息从第一个人传到最后一个人，每经过一个人就可能失真一点。如果队伍很长，最后的信息可能完全走样。这就是深层网络的"梯度消失"问题。残差连接就像给队伍旁边修了一条"高速公路"——原始信息可以直接传到后面，不会被稀释。

残差网络上开创性的论文，取得了巨大成功，论文至今已经被引用了超过了31万次，但站在今天看，残差存在以下局限性：

- **固定权重 (Fixed Weight)**：残差分支和恒等分支的权重固定为 1:1，无法根据输入动态调整。这限制了模型的表达能力

2024年，字节发表了Hyper Connection的论文，提出将输入到Attention/FFN的原始数据进行拆分，从而解决因为残差分支权重总是1，导致模型表达能力受限的问题。具象化的表达，如果说残差连接是一个只有两个固定音量旋钮的调音台，那Hyper Connection 就是一个智能调音台——它能根据每首歌的风格，自动调节每个通道的音量。有的段落需要突出主旋律（残差分支），有的段落需要保留原声（恒等分支），HC 让网络自己学会调节。

![hyper connection](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/hyper%20connection.png)

但是在实际训练过程中，HC会出现信号放大倍数过大导致训练崩溃的问题。

DeepSeek V4 采用了改进版的 Hyper Connection (mHC)，全称是 **Manifold-Constrained Hyper-Connections（流形约束超连接）**，在 HC 的基础上做了关键优化：
采用**双随机矩阵约束（Doubly Stochastic Constraint）**：mHC 的核心创新，通过 **Sinkhorn-Knopp 算法**，将学习到的矩阵投影到双随机矩阵流形上，它保证了每一行/列的和都是1，这相当于一个“**能量守恒**”定律，确保了信号在跨层混合时，其**总量始终保持稳定**，从根本上杜绝了原始HC中信号失控（爆炸或消失）的问题。


# 其他
此外，DeepSeek V4还有很多优化，就不逐一列举了
- Muon 优化器通过 Newton-Schulz 迭代对梯度更新矩阵进行正交化，将所有奇异值归一化到接近 1，以实现更均衡、更快的收敛；AdamW 则继续用于 embedding 和少数其他模块。  
- FP4 量化感知训练（QAT）应用于 MoE 专家权重和 CSA indexer，并采用无损的 FP4→FP8 反量化，无需任何修改即可直接复用现有 FP8 流水线。  
- 预训练使用了 32T+ tokens，并采用从 4K 逐步增长到 1M 的序列长度调度，同时借助 Anticipatory Routing 和 SwiGLU Clamping 来抑制损失尖峰。  
- 后训练分为两个阶段：先进行专家训练（每个领域的 SFT + GRPO），再通过 On-Policy Distillation (OPD) 将所有专家合并为一个统一的学生模型。

DeepSeek V4的文档中坦诚地表示比国外SOTA的闭源模型还存在差距，并不是当下最强的模型，为了保证长上下文任务时的处理效率，架构设计也相对复杂。但是，DeepSeek V4在训练和推理阶段同步在国产算力平台上构建，后续随着国产算力生态的成熟与产业协同的深化，其潜力将进一步释放，为中国AI产业走出一条“全栈自主”的道路奠定坚实基础。

>We validated the fine-grained EP scheme on both NVIDIA GPUs and **HUAWEI Ascend** NPUs platforms.
