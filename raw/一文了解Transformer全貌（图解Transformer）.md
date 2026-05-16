---
title: "一文了解Transformer全貌（图解Transformer）"
source: "https://www.zhihu.com/tardis/zm/art/600773858"
author:
published:
created: 2026-04-12
description:
tags:
  - "transformer"
---
390 赞同

20 评论

1315 收藏

打个小广告 ☻，知乎专栏 **《大模型前沿应用》** 的内容已经收录在新书 **《揭秘大模型：从原理到实战》** 中。感兴趣的朋友可以购买，多谢支持！♥♥

![](https://pica.zhimg.com/v2-ffab9c6d92e840c345ed415f1d4c77ec_720w.jpg?source=b555e01d)

揭秘大模型：从原理到实战

京东

![](https://pica.zhimg.com/v2-ffab9c6d92e840c345ed415f1d4c77ec_720w.jpg?source=b555e01d)

揭秘大模型：从原理到实战

京东

---

自2017年Google推出Transformer以来，基于其架构的语言模型便如雨后春笋般涌现，其中Bert、T5等备受瞩目，而近期风靡全球的大模型ChatGPT和LLaMa更是大放异彩。网络上关于Transformer的解析文章非常大，但本文将力求用浅显易懂的语言，为大家深入解析Transformer的技术内核。

## 前言

Transformer是谷歌在2017年的论文《Attention Is All You Need》中提出的，用于NLP的各项任务，现在是谷歌云TPU推荐的参考模型。网上有关Transformer原理的介绍很多，在本文中我们将尽量模型简化，让普通读者也能轻松理解。

## 1\. Transformer整体结构

在机器翻译中，Transformer可以将一种语言翻译成另一种语言，如果把Transformer看成一个黑盒，那么其结构如下图所示：

![](https://pica.zhimg.com/v2-8caa4fff3ca2ce5dc5a29b03fd617ac4_r.jpg)

将法语翻译成英语

那么拆开这个黑盒，那么可以看到Transformer由若干个编码器和解码器组成，如下图所示：

![](https://pic1.zhimg.com/v2-73b11f65411c0cdfecca80585b7a31b2_r.jpg)

继续将Encoder和Decoder拆开，可以看到完整的结构，如下图所示：

![](https://pic4.zhimg.com/v2-fc13ed90487a03daad57497e8028a093_r.jpg)

Transformer整体结构（引自谷歌论文）

可以看到Encoder包含一个Muti-Head Attention模块，是由多个Self-Attention组成，而Decoder包含两个Muti-Head Attention。Muti-Head Attention上方还包括一个 Add & Norm 层，Add 表示残差连接 (Residual Connection) 用于防止网络退化，Norm 表示 Layer Normalization，用于对每一层的激活值进行归一化。

假设我们的输入包含两个单词，我们看一下Transformer的整体结构：

![](https://picx.zhimg.com/v2-8737942379ca4f57b828e8a856de79fb_r.jpg)

Transformer整体结构（输入两个单词的例子）

为了能够对Transformer的流程有个大致的了解，我们举一个简单的例子，还是以之前的为例，将法语"Je suis etudiant"翻译成英文。

- **第一步** ：获取输入句子的每一个单词的表示向量 ， 由单词的Embedding和单词位置的Embedding 相加得到。
![](https://pic1.zhimg.com/v2-1745b50be609efad5090fa5801954f6c_r.jpg)

Transformer输入表示

- **第二步** ：将单词向量矩阵传入Encoder模块，经过N个Encoder后得到句子所有单词的编码信息矩阵 ，如下图。输入句子的单词向量矩阵用 表示，其中 是单词个数， 表示向量的维度（论文中 ）。每一个Encoder输出的矩阵维度与输入完全一致。
![](https://pic2.zhimg.com/v2-750502dbcc93b061723262afdabd5a39_r.jpg)

输入X经过Encoder输出编码矩阵C

- **第三步** ：将Encoder输出的编码矩阵 传递到Decoder中，Decoder会根据当前翻译过的单词 翻译下一个单词 ，如下图所示。
![](https://pic1.zhimg.com/v2-1c19890bfb2738610a595a55f473492c_r.jpg)

Transformer Decoder预测

上图Decoder接收了Encoder的编码矩阵，然后首先输入一个开始符 "<Begin>"，预测第一个单词，输出为"I"；然后输入翻译开始符 "<Begin>" 和单词 "I"，预测第二个单词，输出为"am"，以此类推。这是Transformer的大致流程，接下来介绍里面各个部分的细节。

## 2\. Transformer的输入表示

Transformer中单词的输入表示由 **单词Embedding** 和 **位置Embedding** （Positional Encoding）相加得到。

![](https://pic1.zhimg.com/v2-1745b50be609efad5090fa5801954f6c_r.jpg)

Transformer输入表示

### 2.1 单词Embedding

单词的Embedding可以通过Word2vec等模型预训练得到，可以在Transformer中加入Embedding层。

### 2.2 位置Embedding

Transformer 中除了单词的Embedding，还需要使用位置Embedding 表示单词出现在句子中的位置。 **因为 Transformer不采用RNN结构，而是使用全局信息，不能利用单词的顺序信息，而这部分信息对于NLP来说非常重要。** 所以Transformer中使用位置Embedding保存单词在序列中的相对或绝对位置。

位置Embedding用 表示， 的维度与单词Embedding相同。 可以通过训练得到，也可以使用某种公式计算得到。在Transformer中采用了后者，计算公式如下：

![\begin{align} &PE_{pos,2i}=sin\left( pos/10000^{2i/d_{model}} \right)\\ &PE_{pos,2i+1}=cos\left( pos/10000^{2i/d_{model}} \right) \end{align}\tag1](https://www.zhihu.com/equation?tex=%5Cbegin%7Balign%7D+%26PE_%7Bpos%2C2i%7D%3Dsin%5Cleft%28+pos%2F10000%5E%7B2i%2Fd_%7Bmodel%7D%7D+%5Cright%29%5C%5C+%26PE_%7Bpos%2C2i%2B1%7D%3Dcos%5Cleft%28+pos%2F10000%5E%7B2i%2Fd_%7Bmodel%7D%7D+%5Cright%29+%5Cend%7Balign%7D%5Ctag1&consumer=ZHI_MENG) 其中， 表示单词在句子中的位置， 表示 的维度。

## 3\. Multi-Head Attention（多头注意力机制）

![](https://pica.zhimg.com/v2-fe6a25d5b50af58a9419d8496e5574fc_r.jpg)

Transformer内部结构

上图是Transformer的内部结构，其中红色方框内为 **Multi-Head Attention** ，是由多个 **Self-Attention** 组成，具体结构如下图：

![](https://pic3.zhimg.com/v2-c0f64bb363bc16ce1bc091c76399564e_r.jpg)

Self-Attention和Multi-Head Attention

因为 **Self-Attention** 是Transformer的重点，所以我们重点关注 Multi-Head Attention 以及 Self-Attention，首先介绍下Self-Attention的内部逻辑。

### 3.1 Self-Attention结构

![](https://pic4.zhimg.com/v2-e1b03d10b7beb28985fd918dd8dbd6d5_1440w.webp?consumer=ZHI_MENG)

Self-Attention结构

上图是Self-Attention结构，最下面是 (查询)、 (键值)、 (值)矩阵，是通过输入矩阵 和权重矩阵 相乘得到的。

![](https://pic2.zhimg.com/v2-47ae22cfc369bb8ada48e8cd87e6a40b_1440w.webp?consumer=ZHI_MENG)

Q,K,V的计算

得到 之后就可以计算出Self-Attention的输出，如下图所示：

![](https://pic1.zhimg.com/v2-26cf164e2ce740a7cc6afc1e237d88be_r.jpg)

Self-Attention输出

### 3.2 Multi-Head Attention输出

在上一步，我们已经知道怎么通过Self-Attention计算得到输出矩阵 ，而Multi-Head Attention是由多个Self-Attention组合形成的，下图是论文中Multi-Head Attention的结构图。

![](https://picx.zhimg.com/v2-c4e7695fdbf5c46bd45f9428b342a755_r.jpg)

Multi-Head Attention

从上图可以看到Multi-Head Attention包含多个Self-Attention层，首先将输入 分别传递到 个不同的Self-Attention中，计算得到 个输出矩阵 。下图是 的情况，此时会得到 8 个输出矩阵 。

![](https://pic4.zhimg.com/v2-19ffed2e3c0a93608b7837f369e6eed7_r.jpg)

多个Self-Attention

得到8个输出矩阵 后，Multi-Head Attention将它们拼接在一起（Concat），然后传入一个Linear层，得到Multi-Head Attention最终的输出矩阵 。

![](https://pic4.zhimg.com/v2-dad82b6bd8c4a0dc849e58a706049be1_r.jpg)

Multi-Head Attention输出

## 4\. 编码器Encoder结构

![](https://picx.zhimg.com/v2-cefd3fb99de81b635ff3b6aca1af67cd_r.jpg)

Transformer Encoder模块

上图红色部分是Transformer的Encoder结构， 表示Encoder的个数，可以看到是由Multi-Head Attention、Add & Norm、Feed Forward、Add & Norm组成的。前面已经介绍了Multi-Head Attention的计算过程，现在了解一下Add & Norm和 Feed Forward部分。

### 4.1 单个Encoder输出

**Add & Norm** 是指残差连接后使用LayerNorm，表示如下：

![\text{Add & Norm}:\text{LayerNorm}\left( \bm{X}+\text{Sublayer}\left( \bm{X} \right) \right)\tag2](https://www.zhihu.com/equation?tex=%5Ctext%7BAdd+%26+Norm%7D%3A%5Ctext%7BLayerNorm%7D%5Cleft%28+%5Cbm%7BX%7D%2B%5Ctext%7BSublayer%7D%5Cleft%28+%5Cbm%7BX%7D+%5Cright%29+%5Cright%29%5Ctag2&consumer=ZHI_MENG) 其Sublayer表示经过的变换，比如第一个Add & Norm中Sublayer表示Multi-Head Attention。

**Feed Forward** 是指全连接层，表示如下：

因此输入矩阵 经过一个Encoder后，输出表示如下：

![\begin{align}  &\bm{O}=\text{LayerNorm}\left( \bm{X}+\text{Multi-Head-Attention}\left( \bm{X} \right) \right)\\ &\bm{O}=\text{LayerNorm}\left( \bm{O}+\text{FFN}\left( \bm{O} \right) \right) \end{align} \tag4](https://www.zhihu.com/equation?tex=%5Cbegin%7Balign%7D++%26%5Cbm%7BO%7D%3D%5Ctext%7BLayerNorm%7D%5Cleft%28+%5Cbm%7BX%7D%2B%5Ctext%7BMulti-Head-Attention%7D%5Cleft%28+%5Cbm%7BX%7D+%5Cright%29+%5Cright%29%5C%5C+%26%5Cbm%7BO%7D%3D%5Ctext%7BLayerNorm%7D%5Cleft%28+%5Cbm%7BO%7D%2B%5Ctext%7BFFN%7D%5Cleft%28+%5Cbm%7BO%7D+%5Cright%29+%5Cright%29+%5Cend%7Balign%7D+%5Ctag4&consumer=ZHI_MENG)

### 4.2 多个Encoder输出

通过上面的单个Encoder，输入矩阵 ，最后输出矩阵 。通过多个Encoder叠加，最后便是编码器Encoder的输出。

## 5\. 解码器Decoder结构

![](https://pica.zhimg.com/v2-7b3d618fa051cc60576154b949b613c6_r.jpg)

Transformer Decoder模块

上图红色部分为Transformer的Decoder结构，与Encoder相似，但是存在一些区别：

- 包含两个Multi-Head Attention
- 第一个Multi-Head Attention采用了Masked操作
- 第二个Multi-Head Attention的 矩阵使用Encoder的 **编码信息矩阵** 进行计算，而 使用上一个 Decoder的输出计算
- 最后有一个Softmax层计算下一个翻译单词的概率

### 5.1 第一个Multi-Head Attention

Decoder的第一个Multi-Head Attention采用了Masked操作，因为在翻译的过程中是顺序翻译的，即翻译完第 个单词，才可以翻译第 个单词。通过 Masked 操作可以防止第 个单词知道 个单词之后的信息。下面以法语"Je suis etudiant"翻译成英文"I am a student"为例，了解一下 Masked 操作。

在Decoder的时候，需要根据之前翻译的单词，预测当前最有可能翻译的单词，如下图所示。首先根据输入"<Begin>"预测出第一个单词为"I"，然后根据输入"<Begin> I" 预测下一个单词 "am"。

![](https://pic3.zhimg.com/v2-d381d89dc81313a26717bf72dbda4294_r.jpg)

Decoder预测（右图有问题，应该是Decoder 1）

Decoder在预测第 个输出时，需要将第 之后的单词掩盖住， **Mask操作是在Self-Attention的Softmax之前使用的，** 下面以前面的"I am a student"为例。

**第一步：** 是Decoder的输入矩阵和 **Mask** 矩阵，输入矩阵包含"<Begin> I am a student"4个单词的表示向量， **Mask** 是一个 的矩阵。在 **Mask** 可以发现单词"<Begin>"只能使用单词"<Begin>"的信息，而单词"I"可以使用单词"<Begin> I"的信息，即只能使用之前的信息。

![](https://pic2.zhimg.com/v2-a52bc9153f72664214753253acbaa3e9_r.jpg)

输入矩阵与Mask矩阵

**第二步** ：接下来的操作和之前Encoder中的Self-Attention一样，只是在Softmax之前需要进行Mask操作。

![](https://pic4.zhimg.com/v2-aa7b81c5ef8e153affd45d78c484f0e3_r.jpg)

Mask Self-Attention输出

**第三步** ：通过上述步骤就可以得到一个Mask Self-Attention的输出矩阵，然后和Encoder类似，通过Multi-Head Attention拼接多个输出然后计算得到第一个Multi-Head Attention的输出 ， 与输入 维度一样。

### 5.2 第二个Multi-Head Attention

Decoder的第二个Multi-Head Attention变化不大， 主要的区别在于其中Self-Attention的 矩阵不是使用上一个Multi-Head Attention的输出，而是使用 **Encoder的编码信息矩阵** 计算的。根据Encoder的输出 计算得到 ，根据上一个Multi-Head Attention的输出 计算 。这样做的好处是在Decoder的时候，每一位单词（这里是指"I am a student"）都可以利用到Encoder所有单词的信息（这里是指"Je suis etudiant"）。

## 6\. Softmax预测输出

![](https://pic4.zhimg.com/v2-7f93a08d8b68fda543bb93ef4fa5d045_r.jpg)

Softmax预测输出

编码器Decoder最后的部分是利用 Softmax 预测下一个单词，在Softmax之前，会经过Linear变换，将维度转换为词表的个数。

假设我们的词表只有6个单词，表示如下：

![](https://pic3.zhimg.com/v2-a42a9f12991d9e57c8e3b7c484211f24_r.jpg)

词表

因此，最后的输出可以表示如下：

![](https://pic1.zhimg.com/v2-b47a5cdc84b9d3fa78db9ff5bf51909e_r.jpg)

Softmax预测输出示例

## 总结

Transformer由于可并行、效果好等特点，如今已经成为机器翻译、特征抽取等任务的基础模块，目前ChatGPT特征抽取的模块用的就是Transformer，这对于后面理解ChatGPT的原理做了好的铺垫。

## 代码实现

[绝密伏击：OPenAI ChatGPT（一）：Tensorflow实现Transformer](https://zhuanlan.zhihu.com/p/603243890)

## 参考

[初识CV：Transformer模型详解（图解最完整版）](https://zhuanlan.zhihu.com/p/338817680)

[数据汪：BERT大火却不懂Transformer？读这一篇就够了](https://zhuanlan.zhihu.com/p/54356280)

[The Illustrated Transformer](https://link.zhihu.com/?target=https%3A//jalammar.github.io/illustrated-transformer/)

[忆臻：搞懂Transformer结构，看这篇PyTorch实现就够了（上）](https://zhuanlan.zhihu.com/p/48731949)

[The Annotated Transformer](https://link.zhihu.com/?target=https%3A//nlp.seas.harvard.edu/2018/04/03/attention.html)

[arxiv.org/pdf/1706.0376](https://link.zhihu.com/?target=https%3A//arxiv.org/pdf/1706.03762.pdf)

[青空栀浅：图解Transformer](https://zhuanlan.zhihu.com/p/69973607)

[Ph0en1x：Transformer结构及其应用详解--GPT、BERT、MT-DNN、GPT-2](https://zhuanlan.zhihu.com/p/69290203)

[大师兄：ChatGPT/InstructGPT详解](https://zhuanlan.zhihu.com/p/590311003)

[张俊林：ChatGPT会取代搜索引擎吗](https://zhuanlan.zhihu.com/p/589533490)

[张俊林：放弃幻想，全面拥抱Transformer：自然语言处理三大特征抽取器（CNN/RNN/TF）比较](https://zhuanlan.zhihu.com/p/54743941)

编辑于 2025-09-27 · 著作权归作者所有