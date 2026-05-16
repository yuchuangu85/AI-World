---
title: "【超详细】【原理篇&实战篇】一文读懂Transformer-CSDN博客"
source: "https://blog.csdn.net/weixin_42475060/article/details/121101749"
author:
  - "[[成就一亿技术人!]]"
  - "[[hope_wisdom 发出的红包]]"
published:
created: 2026-04-12
description: "文章浏览阅读10w+次，点赞2.5k次，收藏9.1k次。本文详细介绍了Transformer模型，一种用于自然语言处理的深度学习架构。涵盖了自注意力机制、多头注意力、位置编码等核心概念，并提供了实现代码。"
tags:
  - "transformer"
---
---

## 前言

Transformer是一种用于自然语言处理（NLP）和其他序列到序列（sequence-to-sequence）任务的深度学习模型架构，它在2017年由Vaswani等人首次提出。Transformer架构引入了自注意力机制（self-attention mechanism），这是一个关键的创新，使其在处理序列数据时表现出色。

<iframe frameborder="0" src="https://live.csdn.net/v/embed/281649" allowfullscreen="true"></iframe>

## 一、Transformer是什么？

> Transformer是一种用于自然语言处理（NLP）和其他序列到序列（sequence-to-sequence）任务的深度学习模型架构，它在2017年由Vaswani等人首次提出。Transformer架构引入了自注意力机制（self-attention mechanism），这是一个关键的创新，使其在处理序列数据时表现出色。  
> **以下是Transformer的一些重要组成部分和特点** ：
> 
> - 自注意力机制（Self-Attention）：这是Transformer的核心概念之一，它使模型能够同时考虑输入序列中的所有位置，而不是像循环神经网络（RNN）或卷积神经网络（CNN）一样逐步处理。自注意力机制允许模型根据输入序列中的不同部分来赋予不同的注意权重，从而更好地捕捉语义关系。
> - 多头注意力（Multi-Head Attention）：Transformer中的自注意力机制被扩展为多个注意力头，每个头可以学习不同的注意权重，以更好地捕捉不同类型的关系。多头注意力允许模型并行处理不同的信息子空间。
> - 堆叠层（Stacked Layers）：Transformer通常由多个相同的编码器和解码器层堆叠而成。这些堆叠的层有助于模型学习复杂的特征表示和语义。
> - 位置编码（Positional Encoding）：由于Transformer没有内置的序列位置信息，它需要额外的位置编码来表达输入序列中单词的位置顺序。
> - 残差连接和层归一化（Residual Connections and Layer Normalization）：这些技术有助于减轻训练过程中的梯度消失和爆炸问题，使模型更容易训练。
> - 编码器和解码器：Transformer通常包括一个编码器用于处理输入序列和一个解码器用于生成输出序列，这使其适用于序列到序列的任务，如机器翻译。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4139e5756879784f0604d1c420e50a86.png#pic_center)

### 1-1、 Transformer的结构：

**Nx = 6，Encoder block由6个encoder堆叠而成，图中的一个框代表的是一个encoder的内部结构，一个Encoder是由Multi-Head Attention和全连接神经网络Feed Forward Network构成。如下图所示** ：  
  
*简略结构（每一个编码器都对应上图的一个encoder结构）：*  
  
*Transformer的编码组件是由6个编码器叠加在一起组成的，解码器同样如此。所有的编码器在结构上是相同的，但是它们之间并没有共享参数。编码器的简略结构如下：*  
  
**从编码器输入的句子首先会经过一个自注意力层，这一层帮助编码器在对每个单词编码的时候时刻关注句子的其它单词。解码器中的解码注意力层的作用是关注输入句子的相关部分，类似于seq2seq的注意力。原结构中使用到的是多头注意力机制（Multi-Head Attention），我们先从基础——自注意力机制开始讲起：**

### 1-2、 自注意力机制

**自注意力的作用** ：随着 模型 处理输入序列的每个单词，自注意力会关注整个输入序列的所有单词，帮助模型对本单词更好地进行编码。在处理过程中，自注意力机制会将对所有相关单词的理解融入到我们正在处理的单词中。更具体的功能如下：

- 序列建模：自注意力可以用于序列数据（例如文本、时间序列、音频等）的建模。它可以捕捉序列中不同位置的依赖关系，从而更好地理解上下文。这对于机器翻译、文本生成、情感分析等任务非常有用。
- 并行计算：自注意力可以并行计算，这意味着可以有效地在现代硬件上进行加速。相比于RNN和CNN等序列模型，它更容易在GPU和TPU等硬件上进行高效的训练和推理。（ **因为在自注意力中可以并行的计算得分** ）
- 长距离依赖捕捉：传统的循环神经网络（RNN）在处理长序列时可能面临梯度消失或梯度爆炸的问题。自注意力可以更好地处理长距离依赖关系，因为它不需要按顺序处理输入序列。

**自注意力的结构如下所示** ：  

**自注意力的计算** ：从每个编码器的输入向量（每个单词的词向量，即Embedding，可以是任意形式的词向量，比如说word2vec，GloVe，one-hot编码）  
  
中生成三个向量，即 **查询向量、键向量和一个值向量** 。（这三个向量是通过词嵌入与三个权重矩阵即相乘后创建出来的）新向量在维度上往往比词嵌入向量更低。（512->64）  
  
更一般的，将以上所得到的查询向量、键向量、值向量组合起来就可以得到三个向量矩阵Query、Keys、Values。  

**查询向量、键向量和值向量**

计算自注意力的 **第二步** 是计算得分，假设我们在为这个例子中的第一个词“Thinking”计算自注意力向量，我们需要拿输入句子中的每个单词对“Thinking”打分。这些分数是通过所有输入句子的单词的键向量与“Thinking”的查询向量相点积来计算的。  
  
**第三步和第四步** 是将分数除以8(8是论文中使用的键向量的维数64的平方根，这会让梯度更稳定。这里也可以使用其它值，8只是默认值，这样做是为了防止内积过大。)，然后通过softmax传递结果。随着模型处理输入序列的每个单词，自注意力会关注整个输入序列的所有单词，帮助模型对本单词更好地进行编码。softmax的作用是使所有单词的分数归一化，得到的分数都是正值且和为1。  
  
这个softmax分数决定了每个单词对编码当下位置（“Thinking”）的贡献。显然，已经在这个位置上的单词将获得最高的softmax分数，  
**第五步** 是将每个值向量乘以softmax分数(这是为了准备之后将它们求和)。这里的直觉是希望关注语义上相关的单词，并弱化不相关的单词(例如，让它们乘以0.001这样的小数)。  
**Softmax函数** ：或称归一化指数函数，它将每一个元素的范围都压缩到（0，1）之间，并且所有元素的和为1。  
**第六步** 是对加权值向量求和，然后即得到自注意力层在该位置的输出(在我们的例子中是对于第一个单词)。  
  
整体的计算图如图所示：  
  
最终得到了自注意力，并将得到的向量传递给前馈 神经网络 。以上二到六步合为一个公式计算自注意力层的输出。

**自注意力层的完善——“多头”注意力机制：**

对应整体结构图中的 **Multi——Head Attention**  
1、扩展了模型专注于不同位置的能力。  
2、有多个查询/键/值权重矩阵集合，（Transformer使用八个注意力头）并且每一个都是随机初始化的。和上边一样，用矩阵X乘以WQ、WK、WV来产生查询、键、值矩阵。  
3、 **self-attention** 只是使用了一组WQ、WK、WV来进行变换得到查询、键、值矩阵，而Multi-Head Attention使用多组WQ，WK，WV得到多组查询、键、值矩阵，然后每组分别计算得到一个Z矩阵。  
  
前馈层只需要一个矩阵，则把得到的8个矩阵拼接在一起，然后用一个附加的权重矩阵与它们相乘。  
  
**总结整个流程：**  
  
**编码it一词时，不同注意力的头集中在哪里，当我们编码it这个单词时：（图中只列举出了两个注意力）：**

- 其中一个注意力头集中在The animal
- 另一个注意力头集中在tire上。即形象解释it代指的是animal和tire。

### 1-3、使用位置编码表示序列的顺序

**为什么要用位置编码？**

- 如果不添加位置编码，那么无论单词在什么位置，它的注意力分数都是确定的。这不是我们想要的。
- 为了理解单词顺序，Transformer为每个输入的词嵌入添加了一个向量，这样能够更好的表达词与词之间的关系。词嵌入与位置编码相加，而不是拼接，他们的效率差不多，但是拼接的话维度会变大，所以不考虑。（这里位置向量如何得到，以哪种计算方法得到，以及词嵌入与位置编码如何结合是可以尝试实验的点，可以看以下链接思考这个问题）。 [如何理解Transformer论文中的positional encoding，和三角函数有什么关系？](https://www.zhihu.com/question/347678607).  
	  
	为了让模型理解单词的顺序，我们添加了位置编码向量，这些向量的值遵循特定的模式。

### 1-4、 Add &Normalize

在经过多头 注意力机制 得到矩阵Z之后，并没有直接传入全连接神经网络，而是经过了一步Add&Normalize。  
  
Add & Norm 层由 Add 和 Norm 两部分组成，其计算公式如下：  
  
其中 X表示 Multi-Head Attention 或者 Feed Forward 的输入，MultiHeadAttention(X) 和 FeedForward(X) 表示输出 (输出与输入 X 维度是一样的，所以可以相加)。

- **Add**  
	Add，就是在z的基础上加了一个残差块X，加入残差块的目的是为了防止在深度神经网络的训练过程中发生退化的问题，退化的意思就是深度神经网络通过增加网络的层数，Loss逐渐减小，然后趋于稳定达到饱和，然后再继续增加网络层数，Loss反而增大。
- **ResNet残差神经网络**  
	为了了解残差块，我们引入ResNet残差神经网络，神经网络退化指的是在达到最优网络层数之后，神经网络还在继续训练导致Loss增大，对于多余的层，我们需要保证多出来的网络进行恒等映射。只有进行了恒等映射之后才能保证这多出来的神经网络不会影响到模型的效果。 **残差连接主要是为了防止网络退化。**  
	  
	上图就是构造的一个残差块，X是输入值，F（X）是经过第一层线性变换后并且激活的输出，在第二层线性变化之后，激活之前，F（X）加入了这一层输入值X，然后再进行激活后输出。

要恒等映射，我们只需要让F（X）=0就可以了。x经过 线性 变换（随机初始化权重一般偏向于0），输出值明显会偏向于0，而且经过激活函数Relu会将负数变为0，过滤了负数的影响。  
这样当网络自己决定哪些网络层为冗余层时，使用ResNet的网络很大程度上解决了学习恒等映射的问题，用学习残差F(x)=0更新该冗余层的参数来代替学习h(x)=x更新冗余层的参数。

- **Normalize**  
	归一化目的：  
	1、加快训练速度  
	2、提高训练的稳定性  
	使用到的归一化方法是Layer Normalization。  
	  
	LN是在同一个样本中不同神经元之间进行归一化，而BN是在同一个batch中不同样本之间的同一位置的神经元之间进行归一化。  
	BN是对于相同的维度进行归一化，但是咱们NLP中输入的都是词向量，一个300维的词向量，单独去分析它的每一维是没有意义地，在每一维上进行归一化也是适合地，因此这里选用的是LN。

### 1-5、全连接层Feed Forward

  
全连接层是一个两层的神经网络，先线性变换，然后ReLU非线性，再线性变换。  
这两层网络就是为了将输入的Z映射到更加高维的空间中然后通过非线性函数ReLU进行筛选，筛选完后再变回原来的维度。  
经过6个encoder后输入到decoder中。

### 1-6、Decoder整体结构

  
**和Encoder Block一样，Decoder也是由6个decoder堆叠而成的，Nx=6。包含两个 Multi-Head Attention 层。第一个 Multi-Head Attention 层采用了 Masked 操作。第二个 Multi-Head Attention 层的K, V矩阵使用 Encoder 的编码信息矩阵C进行计算，而Q使用上一个 Decoder block 的输出计算。**

**Masked Multi-Head Attention**  
与Encoder的Multi-Head Attention计算原理一样，只是多加了一个mask码。mask 表示掩码，它对某些值进行掩盖，使其在参数更新时不产生效果。Transformer 模型里面涉及两种 mask，分别是 padding mask 和 sequence mask。为什么需要添加这两种mask码呢？

- **padding mask**  
	什么是 padding mask 呢？因为每个批次输入序列长度是不一样的也就是说，我们要对输入序列进行对齐。具体来说，就是给在较短的序列后面填充 0。但是如果输入的序列太长，则是截取左边的内容，把多余的直接舍弃。因为这些填充的位置，其实是没什么意义的，所以我们的attention机制不应该把注意力放在这些位置上，所以我们需要进行一些处理。  
	具体的做法是，把这些位置的值加上一个非常大的负数(负无穷)，这样的话，经过 softmax，这些位置的概率就会接近0！
- **sequence mask**  
	sequence mask 是为了使得 decoder 不能看见未来的信息。对于一个序列，在 time\_step 为 t 的时刻，我们的解码输出应该只能依赖于 t 时刻之前的输出，而不能依赖 t 之后的输出。因此我们需要想一个办法，把 t 之后的信息给隐藏起来。这在训练的时候有效，因为训练的时候每次我们是将target数据完整输入进decoder中地，预测时不需要，预测的时候我们只能得到前一时刻预测出的输出。  
	那么具体怎么做呢？也很简单：产生一个上三角矩阵，上三角的值全为0。把这个矩阵作用在每一个序列上，就可以达到我们的目的。

**注意：  
1、在Encoder中的Multi-Head Attention也是需要进行mask的，只不过Encoder中只需要padding mask即可，而Decoder中需要padding mask和sequence mask。**  
**2、Encoder中的Multi-Head Attention是基于Self-Attention地，Decoder中的第二个Multi-Head Attention就只是基于Attention，它的输入Quer来自于Masked Multi-Head Attention的输出，Keys和Values来自于Encoder中最后一层的输出。**

### 1-7、输出

Output如图中所示，首先经过一次线性变换（线性变换层是一个简单的全连接神经网络，它可以把解码组件产生的向量投射到一个比它大得多的，被称为对数几率的向量里），然后Softmax得到输出的概率分布（softmax层会把向量变成概率），然后通过词典，输出概率最大的对应的单词作为我们的预测输出。  

### 1-8、transformer的优缺点：

优点：  
1、效果好  
2、可以并行训练，速度快  
3、很好的解决了长距离依赖的问题  
缺点：  
完全基于self-attention，对于词语位置之间的信息有一定的丢失，虽然加入了positional encoding来解决这个问题，但也还存在着可以优化的地方。

## 二、Self-Attention的实现

### 2-0、过程

- 准备输入
- 初始化参数
- 获取key，query和value
- 给input1计算attention score
- 计算softmax
- 给value乘上score
- 给value加权求和获取output1
- 重复步骤4-7，获取output2，output3

### 2-1、准备输入（词嵌入向量）

```c
import torch
x = [
  [1, 0, 1, 0], # Input 1
  [0, 2, 0, 2], # Input 2
  [1, 1, 1, 1]  # Input 3
 ]
x = torch.tensor(x, dtype=torch.float32)
x
c运行12345678
```

**输出** ：  
*tensor(\[\[1., 0., 1., 0.\],  
\[0., 2., 0., 2.\],  
\[1., 1., 1., 1.\]\])*

### 2-2、初始化参数（Q、K、V矩阵）

**Note** ： Q、K、V矩阵在神经网络初始化的过程中，一般都是随机采样完成并且比较小，可以根据想要输出的维度来确定 Q、K、V矩阵的维度。

```c
w_key = [
  [0, 0, 1],
  [1, 1, 0],
  [0, 1, 0],
  [1, 1, 0]
]
w_query = [
  [1, 0, 1],
  [1, 0, 0],
  [0, 0, 1],
  [0, 1, 1]
]
w_value = [
  [0, 2, 0],
  [0, 3, 0],
  [1, 0, 3],
  [1, 1, 0]
]
w_key = torch.tensor(w_key, dtype=torch.float32)
w_query = torch.tensor(w_query, dtype=torch.float32)
w_value = torch.tensor(w_value, dtype=torch.float32)

print("Weights for key: \n", w_key)
print("Weights for query: \n", w_query)
print("Weights for value: \n", w_value)
c运行12345678910111213141516171819202122232425
```

**输出** ：  
*Weights for key:  
tensor(\[\[0., 0., 1.\],  
\[1., 1., 0.\],  
\[0., 1., 0.\],  
\[1., 1., 0.\]\])  
Weights for query:  
tensor(\[\[1., 0., 1.\],  
\[1., 0., 0.\],  
\[0., 0., 1.\],  
\[0., 1., 1.\]\])  
Weights for value:  
tensor(\[\[0., 2., 0.\],  
\[0., 3., 0.\],  
\[1., 0., 3.\],  
\[1., 1., 0.\]\])*

### 2-3、获取key，query和value

```c
keys = x @ w_key
querys = x @ w_query
values = x @ w_value

print("Keys: \n", keys)
# tensor([[0., 1., 1.],
#         [4., 4., 0.],
#         [2., 3., 1.]])

print("Querys: \n", querys)
# tensor([[1., 0., 2.],
#         [2., 2., 2.],
#         [2., 1., 3.]])
print("Values: \n", values)
# tensor([[1., 2., 3.],
#         [2., 8., 0.],
#         [2., 6., 3.]])
c运行1234567891011121314151617
```

**下图为得到的key，query和value** ：  

### 2-4、计算注意力分数

```c
attn_scores = querys @ keys.T
print(attn_scores)
c运行12
```

**输出** ：  
*tensor(\[\[ 2., 4., 4.\],  
\[ 4., 16., 12.\],  
\[ 4., 12., 10.\]\])*

### 2-5、计算softmax

```c
from torch.nn.functional import softmax

attn_scores_softmax = softmax(attn_scores, dim=-1)
print(attn_scores_softmax)
# tensor([[6.3379e-02, 4.6831e-01, 4.6831e-01],
#         [6.0337e-06, 9.8201e-01, 1.7986e-02],
#         [2.9539e-04, 8.8054e-01, 1.1917e-01]])

# 为了使得后续方便，这里简略将计算后得到的分数赋予了一个新的值
# For readability, approximate the above as follows
attn_scores_softmax = [
  [0.0, 0.5, 0.5],
  [0.0, 1.0, 0.0],
  [0.0, 0.9, 0.1]
]
attn_scores_softmax = torch.tensor(attn_scores_softmax)
print(attn_scores_softmax)
c运行1234567891011121314151617
```

**输出** ：  
*tensor(\[\[6.3379e-02, 4.6831e-01, 4.6831e-01\],  
\[6.0337e-06, 9.8201e-01, 1.7986e-02\],  
\[2.9539e-04, 8.8054e-01, 1.1917e-01\]\])  
tensor(\[\[0.0000, 0.5000, 0.5000\],  
\[0.0000, 1.0000, 0.0000\],  
\[0.0000, 0.9000, 0.1000\]\])*

### 2-6、给value乘上score

```c
weighted_values = values[:,None] * attn_scores_softmax.T[:,:,None]
print(weighted_values)
c运行12
```

**输出** ：  
*tensor(\[\[\[0.0000, 0.0000, 0.0000\],  
\[0.0000, 0.0000, 0.0000\],  
\[0.0000, 0.0000, 0.0000\]\],  
\[\[1.0000, 4.0000, 0.0000\],  
\[2.0000, 8.0000, 0.0000\],  
\[1.8000, 7.2000, 0.0000\]\],  
\[\[1.0000, 3.0000, 1.5000\],  
\[0.0000, 0.0000, 0.0000\],  
\[0.2000, 0.6000, 0.3000\]\]\])*

### 2-7、给value加权求和获取output(得到input1的结果向量)

  
**重复步骤4-7，获取到input2、input3的结果向量**

## 三、Transformer代码实现

```c
c运行1
```

**参考文章** ：  
[图解Transformer（完整版）](https://blog.csdn.net/longxinchen_ml/article/details/86533005).  
[史上最小白之Transformer详解](https://blog.csdn.net/Tink1995/article/details/105080033?spm=1001.2014.3001.5501).  
[一文读懂BERT(原理篇)](https://blog.csdn.net/jiaowoshouzi/article/details/89073944/).  
[Transformer模型详解（图解最完整版）](https://zhuanlan.zhihu.com/p/338817680).  
[【从 0 开始学习 Transformer】 上篇：Transformer 搭建与理解](https://editor.csdn.net/md/?articleId=121101749).  
[Transformer 一篇就够了（一）： Self-attenstion](https://zhuanlan.zhihu.com/p/345680792).  
[TRANSFORMERS FROM SCRATCH](https://peterbloem.nl/blog/transformers).  
[【Transformer】10分钟学会Transformer | Pytorch代码讲解 | 代码可运行](https://zhuanlan.zhihu.com/p/403433120).  
[Transformer 模型详解](https://baijiahao.baidu.com/s?id=1651219987457222196).  
[三万字最全解析！从零实现Transformer（小白必会版😃）](https://zhuanlan.zhihu.com/p/648127076).  
[算法工程师当前选哪个方向好？](https://www.zhihu.com/question/398876586/answer/3333520368).  
[Happy-LLM（从0开始的大语言模型原理与实践教程）: https://github.com/datawhalechina/happy-llm?tab=readme-ov-file](https://github.com/datawhalechina/happy-llm?tab=readme-ov-file)

## 总结

以上就是我总结（照抄）其它优秀博主的全部transformer内容了。

微信名片