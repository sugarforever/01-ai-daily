---
title: "GLM-5.2 深度解读：当开源模型真正逼近闭源天花板"
date: 2026-06-19
excerpt: "753B MoE、百万 token 上下文、IndexShare 稀疏注意力——Z.ai 开源 GLM-5.2，凭什么让 Simon Willison 和 Sebastian Raschka 都给出最高评价？"
tags: [GLM-5.2, Z.ai, 开源模型, IndexShare, 稀疏注意力, MoE, 长上下文]
---

几天前 Z.ai 开源了 GLM-5.2，圈内讨论不少。花时间读了 Sebastian Raschka 的架构分析、Simon Willison 的实测体验，加上自己的一些理解，整理成这篇技术分享。

## 一、它为什么值得关注

先说结论：**GLM-5.2 可能是目前最强的纯文本开源权重模型**。

这不是随口说的。Simon Willison 原话如此，Sebastian Raschka 也给出了同样的第一印象。Artificial Analysis 的 Intelligence Index 上，GLM-5.2 以 51 分登顶开源模型榜首，领先 MiniMax-M3（44）、DeepSeek V4 Pro（44）和 Kimi K2.6（43）一个身位。更夸张的是编程能力——Artificial Analysis Coding Index 上 GLM-5.2 拿到 68.8，而 Claude Opus 4.8（max）只有 56.7，**高出超过 10 分**。Code Arena WebDev 排行榜上它排第二，仅次于 Claude Fable 5。

一个开源模型在编程上跑赢顶级闭源模型，这放在一年前是难以想象的。

几个关键参数速览：

| 参数 | 数值 |
|---|---|
| 总参数量 | 753B |
| 激活参数量 | 40B（MoE） |
| 上下文窗口 | 1,000,000 tokens |
| 许可证 | MIT |
| 输入价格 | ~$1.40 / million tokens |
| 输出价格 | ~$4.40 / million tokens |

对比一下：GPT-5.5 是 $5 / $30，Claude Opus 4.5-4.8 是 $5 / $25。GLM-5.2 的价格只有它们的 **1/3 到 1/7**，性能和价格的双重优势让它非常值得关注。

## 二、架构全景：站在巨人肩膀上

Raschka 的文章把架构脉络理得很清楚。GLM-5.2 并不是从零开始设计的——它建立在 GLM-5 和 GLM-5.1 的基础上，同时吸收了两项关键的外部技术。

### Multi-head Latent Attention（MLA）

这个机制最早来自 DeepSeek，核心思路是在注意力计算中对 Key-Value 做**低秩压缩**。简单说，传统注意力需要缓存完整的 K 和 V 矩阵，显存开销巨大；MLA 把它们投影到一个更小的隐空间，推理时只需要缓存压缩后的表示，**大幅降低 KV Cache 的显存占用**。

对于 1M 上下文窗口来说，这个设计几乎是必须的——否则显存会先撑不住。

### DeepSeek Sparse Attention（DSA）

这是 DeepSeek V3.2 引入的稀疏注意力机制。核心思想很简单：不是所有历史 token 都值得关注，DSA 用一个**可学习的 indexer** 为每个 query 动态选择最相关的 top-k 个 token 来参与注意力计算。

注意力从「全连接」变成了「稀疏选择」，计算复杂度大幅降低，理论上可以支持更长的上下文。

## 三、IndexShare：这次真正的亮点

上面说的 MLA 和 DSA 都是已有的技术。GLM-5.2 真正的创新是这个——**IndexShare**。

论文是：[IndexShare: Reducing Token Selection Cost for Long-Context Inference](https://arxiv.org/abs/2503.19637)

DSA 的问题是：每一层都要跑 indexer 来选择 top-k token，这在 deep network（GLM-5.2 有几十层）中成本还是很可观。IndexShare 的思路特别直观：

> **不需要每层都算 indexer，每 4 层算一次就够了，中间层复用选出来的 token 索引。**

Raschka 的原话是：

> "Instead of recomputing the sparse-attention top-k indexer in every layer, GLM-5.2 runs the full indexer only once every four layers. The following layers then reuse the selected token indices."

翻译成大白话：

- 标准 DSA：L1 选 → L1 算 → L2 选 → L2 算 → L3 选 → L3 算 → L4 选 → L4 算……
- IndexShare DSA：L1 选 → L1~L4 算（复用 L1 的索引）→ L5 选 → L5~L8 算……

**计算量直接降到原来的约 1/4，注意力模式仍然是自适应的。**

这个设计非常优雅。它建立在这样一个假设上：相邻层的注意力关注模式是高度相似的，不需要每层都重新做选择。实验数据应该支持这一点，否则性能会掉。

IndexShare 是让 1M 上下文窗口在推理成本上变得可负担的关键工程创新。

## 四、实测表现：不仅是跑分

Simon Willison 的实测给了一个很有意思的侧面印证。他之前用 GLM-5.1 生成过 SVG，效果相当惊艳。这次他用同样的 prompt 测试了 GLM-5.2。

### 🎨 会骑自行车的鹈鹕

Prompt 是 "Generate an SVG of a pelican riding a bicycle"。

> "It's a self-contained fully animated SVG, and the animations aren't broken! Often I'll see eyes falling off or wheels rotating independently of the bicycle but here everything works great."

Simon 特别提到，很多模型生成的动画 SVG 会有"眼睛掉下来"或"车轮和车身分离"这种低级 bug，GLM-5.2 没有这些问题。自行车轮子跟着车身转，不是自己凭空乱转——这种"物理常识"出现在 SVG 生成里，说明模型的推理能力确实强。

可以看看他分享的 SVG 效果：[点此查看](https://gist.github.com/simonw/5c989366b796f054d9ae1ad7e38dc03a)

### 🐭 令人失望的负鼠

但是！同样的模型，换了个 prompt 就翻车了。GLM-5.1 生成的那只"骑着电动滑板车的北弗吉尼亚负鼠"，堪称 Simon 收集中最经典的 SVG 作品——负鼠的表情、细节、CSS 动画轮子，全都无可挑剔。

GLM-5.2 的版本？差得太远。Simon 的原话是 "This is such a step down from GLM-5.1"，而且 5.2 做的 SVG **甚至没有动画**。

这个反例其实挺有意思。**并不是每项能力都线性提升的**，有些任务上老模型可能在某些微妙的地方做得更好。模型更新时永远要测试自己的 use case，不能只看跑分。

## 五、一些值得注意的细节

### 输出 Token 消耗偏高

Artificial Analysis 发现 GLM-5.2 每任务平均输出 43k tokens，比 GLM-5.1 的 26k 高出不少，也高于 MiniMax-M3（24k）和 DeepSeek V4 Pro（37k）。这可能意味着两条路：要么模型倾向于"多写多想"来保证质量，要么还有优化空间。

### 纯文本模型

GLM-5.2 是纯文本模型，视觉能力在 GLM-5V-Turbo 那边，且**那个没有开源权重**。所以它虽然编程很强，但不适合需要理解图片的多模态场景。

### MIT 许可证

采用 MIT 许可证是一个特别积极的信号。自由使用、修改、甚至商用，没有任何限制。Z.ai 在开源态度上做得比很多公司都好。

## 六、总结与展望

GLM-5.2 在我看来代表了开源大模型的一个重要转折点：

1. **架构创新不再是闭源模型的专利**。IndexShare 是一个很干净的工程创新，思路简单但效果显著——这往往是最好的工程。
2. **开源模型在编程和推理上已经可以超越闭源标杆**。Coding Index 上超过 Claude Opus 4.8 这件事，值得反复强调。
3. **价格差距还在拉大**。1/3 到 1/7 的价格 + MIT 许可证，对个人开发者和中小团队来说是非常 real 的好处。
4. **上下文窗口正向百万 token 迈进**。IndexShare 这种跨层复用的思路，可能就是未来长上下文模型的标配。

当然也有不足：纯文本、输出偏长、特定任务可能不如上一代。但这些都不影响它是目前最值得一试的开源模型。

---

> **一些好的阅读起点：**
>
> - 模型权重（HuggingFace）：[zai-org/GLM-5.2](https://huggingface.co/zai-org/GLM-5.2)
> - OpenRouter 在线体验：[z-ai/glm-5.2](https://openrouter.ai/z-ai/glm-5.2)
> - Sebastian Raschka 的架构分析：[GLM-5.2 IndexShare Architecture Note](https://sebastianraschka.com/blog/2026/glm-5-2-indexshare.html)
> - Simon Willison 的上手评测：[GLM-5.2 is probably the most powerful text-only open weights LLM](https://simonwillison.net/2026/Jun/17/glm-52/#atom-everything)
> - IndexShare 论文：[IndexShare: Reducing Token Selection Cost for Long-Context Inference](https://arxiv.org/abs/2503.19637)
