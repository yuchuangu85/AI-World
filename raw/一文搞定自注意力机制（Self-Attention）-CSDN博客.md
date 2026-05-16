---
title: "一文搞定自注意力机制（Self-Attention）-CSDN博客"
source: "https://blog.csdn.net/weixin_42110638/article/details/134016569"
author:
  - "[[成就一亿技术人!]]"
  - "[[hope_wisdom 发出的红包]]"
published:
created: 2026-04-12
description: "文章浏览阅读10w+次，点赞334次，收藏1.4k次。本文深入浅出地介绍了自注意力机制（Self-Attention）的概念、原理及其在自然语言处理中的应用，特别是其在Transformer模型中的关键作用。"
tags:
  - "transformer"
---
自注意力机制（Self-Attention）作为 注意力机制 中的一种，也被称为intra Attention（内部Attention），是大名鼎鼎的Transformer重要组成部分，今天张张将详细 **介绍自注意力机制(Self-Attention)。**

![](https://i-blog.csdnimg.cn/blog_migrate/fae4a0c88137f1e9887b9b1072997916.png)

## 本文目录

1 Self-Attention的概念

2 Self-Attention的原理

3 Self-Attention的作用

4 Self-Attention的问题

---

### 1 Self-Attention的概念

Self-Attention， **自注意力机制，又称内部注意力机制** ，顾名思义，是一种 **将单个序列的不同位置关联起来以计算同一序列的表示的注意机制。**

通过对注意力机制的学习我们知道，在一般任务的Encoder-Decoder框架中，输入Source和输出Target内容是不一样的，比如对于英-中机器翻译来说，Source是英文句子，Target是对应的翻译出的中文句子， **Attention机制发生在Target的元素Query和Source中的所有元素之间（即Attention机制与自身还有关注对象都有关系）** 。

而Self-Attention顾名思义，指的不是Target和Source之间的Attention机制， **而是Source内部元素之间或者Target内部元素之间发生的Attention机制，也可以理解为Target=Source这种特殊情况下的注意力计算机制。（即Self-Attention只关注输入本身or只关注关注对象本身）**

**另一个好的解释就是：**

自注意力机制和注意力机制的区别就在于， **注意力机制的查询和键是不同来源的** ，例如，在Encoder-Decoder模型中，键是Encoder中的元素，而查询是Decoder中的元素。在中译英模型中，查询是中文单词特征，而键则是英文单词特征。 **而自注意力机制的查询和键则都是来自于同一组的元素** ，例如，在Encoder-Decoder模型中，查询和键都是Encoder中的元素，即查询和键都是中文特征，相互之间做注意力汇聚。可以理解为同一句话中的词元或者同一张图像中不同的patch，这都是一组元素内部相互做注意力机制，因此，自注意力机制（self-attention）也被称为内部注意力机制（intra-attention）。  
**优点** ： **可以建立全局的依赖关系，扩大图像的感受野。相比于CNN，其感受野更大，可以获取更多上下文信息。**

缺点： **自注意力机制是通过筛选重要信息，过滤不重要信息实现的，这就导致其有效信息的抓取能力会比CNN小一些。** 这样是因为自注意力机制相比CNN，无法利用图像本身具有的尺度，平移不变性，以及图像的特征局部性（图片上相邻的区域有相似的特征，即同一物体的信息往往都集中在局部）这些先验知识， **只能通过大量数据进行学习。这就导致自注意力机制只有在大数据的基础上才能有效地建立准确的全局关系，而在小数据的情况下，其效果不如CNN。**

**关于自注意力机制对比CNN：**

![](https://i-blog.csdnimg.cn/direct/3ba9b5dc6e4e4261869facd870f2f34e.png)

[CNN与注意力机制的本质区别](https://mp.weixin.qq.com/s/XJy0EMFp7HvwBaOmkn5_Yg "CNN与注意力机制的本质区别")

**重点总结：**

**1）自注意力机制小结：**

**官话：如果查询和键是同一组内的特征，并且相互做注意力机制，则称为自注意力机制或内部注意力机制。而如果查询和键是不同组的特征，并且相互做注意力机制，则是普通的注意力机制。**

**大白话：Self-Attention只关注输入本身or只关注关注对象本身；Attention机制与自身还有关注对象都有关系**

  
**2）多头注意力机制的多头表示对每个Query和所有的Key-Value做多次注意力机制。做两次，就是两头，做三次，就是三头。这样做的意义在于获取每个Query和所有的Key-Value的不同的依赖关系（也就是为了提取不同的特征）（就类似CNN的多个卷积核提取不同的特征，一个意思）。  
3）自注意力机制的优缺点简记为【优点：感受野大，大数据情况下全局建模能力强。缺点：需要很大的数据量】**

---

### 2 Self-Attention的原理

这里先给出Self-Attention的 **架构** 。

![图片](https://i-blog.csdnimg.cn/blog_migrate/af742200d18ec6351932e30e3681fc4f.png)

通过对注意力机制的学习我们知道，对于注意力机制来说，键值对形式的Attention计算公式如下：

上式变换为通用的写法为：

**这就是大名鼎鼎的Attention Fuction** 。在Self-Attention中，公式中的K、Q、V表示如下图所示，可以看出 **QKV的来源都是输入矩阵X与矩阵的乘积，本质上都是X的线性变换（代码上就是分别经过了三个线性层），这也是为什叫做自注意力机制的原因。**

**ps：这三个线性层的输入维度和输出维度的讲究在注意力机制中讲过了哈，不懂得回去看~**

从上式可以看出其计算过程为：首先，计算矩阵Q和K每一行向量的内积， **为了防止内积过大，除以d\_k的平方根** ；其次， **使用Softmax对上述内积的结果进行归一化；** 最后得到Softmax矩阵之后和V相乘，得到最终的输出。

Pytorch示例代码如下：

![](https://i-blog.csdnimg.cn/direct/de9bd7431c3e4a599fbde9cff71650b7.png)

```python
import torch

import torch.nn as nn

import torch.nn.functional as F

import math

 

class SelfAttention(nn.Module):

    def __init__(self, dim_q, dim_k, dim_v):

        super(SelfAttention, self).__init__()

        self.dim_q = dim_q

        self.dim_k = dim_k

        self.dim_v = dim_v

        # 定义线性变换函数

        self.linear_q = nn.Linear(dim_q, dim_k, bias=False)

        self.linear_k = nn.Linear(dim_q, dim_k, bias=False)

        self.linear_v = nn.Linear(dim_q, dim_v, bias=False)

        self._norm_fact = 1 / math.sqrt(dim_k)

 

    def forward(self, x):

        # x: batch, n, dim_q

        # 根据文本获得相应的维度

        batch, n, dim_q = x.shape

        # 如果条件为 True，则程序继续执行；如果条件为 False，则程序抛出一个 AssertionError 异常，并停止执行。

        assert dim_q == self.dim_q  # 确保输入维度与初始化时的dim_q一致

        

        q = self.linear_q(x)  # batch, n, dim_k

        k = self.linear_k(x)  # batch, n, dim_k

        v = self.linear_v(x)  # batch, n, dim_v

        

        # q*k的转置并除以开根号后的dim_k

2

        # 归一化获得attention的相关系数

# batch, n, n

        # attention系数和v相乘，获得最终的得分

        att = torch.bmm(dist, v)

        return att
python运行
```

**关于为什么bias=False：**

- 由于 softmax 函数会将输入转换成一个概率分布，其输出值的范围在0到1之间，并且各输出值的和为1，这减少了偏置项对模型性能的影响。因此，在这种情况下，省略偏置项可以减少模型的参数数量，提高训练速度，简化模型复杂度，并且有助于避免过拟合，提高模型的泛化能力。

**分享一个不错的讲解：**

#### Part1从整体上来看自注意力机制

在详谈自注意力机制之前，最好先从整体把握其运作原理。

举一个简单例子，我们来看下面一个句子。假设我们尝试将它作为输入，并准备翻译这句话。

> ” `The animal didn't cross the street because it was too tired` ”

这句话中的单词"it"指的是什么呢？它是指“street”还是“animal”呢？对于我们人来说，这个问题非常简单。但对一个算法来说，这并不简单。

当模型正在处理单词“it”的时候，自注意力机制允许单词“it”结合单词“animal”一起处理。

也就是说，在注意力及之中，模型可以结合上下文的单词来处理当前单词。从其它单词中找寻“线索”，可以帮助模型更好的编码当前单词。

如果你对RNNs熟悉，思考一下你是如何在处理当前单词时，把处理过的单词的信息保存在hidden state中的。

注意力机制正是如此，通过关联上下文的单词，来辅助处理当前的单词。

![图片](https://i-blog.csdnimg.cn/blog_migrate/0366c95d9d779a2d33659bc12b5f54b9.png)

transformer\_self-attention\_visualization

如上图所示，当我们编码单词"it"时，注意力机制会加强对“The Animal”的注意，来帮助处理当前单词“it”。

#### Part2自注意力机制的细节部分

**在此部分，我们将用向量展示自注意力机制是如何计算的。**

**第一步** ，我们首先用输入的词向量x来生成三个新的向量qkv。每次一个输入的词向量都要生成三个新的向量。这三个新的向量分别叫做，查询向量（ Query ），键向量（Key），值向量（Value）。 **这三个向量是通过输入的词向量x和三个矩阵做点乘得到的。这三个矩阵的权重先随机初始化，之后会在训练过程中调整。**

**三个新向量qkv的维度通常比词向量x的维度要小** 。例如新向量的维度是64，而词向量的输入是512。 **但是，他们不一定必须要比词向量的维度小。**

transformer\_self\_attention\_vectors

如上图所示，和做点乘后，得到单词“ Thinking ”的查询向量。其它向量的计算也同理。

**第二步** ， **用查询向量和健向量做点乘得到注意力Score** 。继续用上图的单词“Thinking”举例子，在 **这一步我们需要计算出基于当前单词“Thinking”的其它所有的单词的Score（也包括当前单词“Thinking”的Score）。在处理当前单词时，Score的值决定了我们需要放多少注意力在相应的其它的输入单词上（即进行相似度计算）。**

通过计算当前单词的查询向量与其它各个单词的键向量的内积，就可以得到Score的值。

由于在这一步中，查询向量和键向量要做点乘处理， **所以查询向量和键向量必须拥有相同的维度（指的是特征维度必须一致，不过有时候就时间维度（序列长度）也就直接都一致了）。**

transformer\_self\_attention\_score

例如，在处理单词“Thinking”时，在其自身“Thinking”上的Score为112，而在单词“Machines”上的Score则为96。

**第三步** ，将得到的Score处以8。为什么是8，而不是其它数字呢？这是因为， **这个数字通常是通过对键向量的维度开方得到的** 。在第一步中，我们假设键向量的维度是64，对其开方，我们就得到了数字8。

第三步的操作会使模型在训练时拥有更稳定的梯度。

**第四步** ， **对第三步中得到的结果进行softmax处理。softmax使得第三步的结果都为正值，并且相加等于一（与注意力机制是一致的）。**

第四步的结果决定了在处理当前单词时，每个单词的重要程度。诚然，对于当前的单词来说，其自身最为重要。但是，这一步也会揪出其它关联性较强的单词。

第三步，第四步的计算过程如下图所示。在这里，我们先将第四步得到的结果叫做soft socre。

self-attention\_softmax

**第五步** ，用每个单词得到的soft score和其值向量的各个元素相乘。在这一步，我们保持那些需要注意的单词的值的完整性。并且，冲淡了那些与当前单词关联性不强的单词，例如softmax socre为0.001的单词。

**第六步** ，把在第五步中得到的向量相加得到最后输出。下图中的即为我们在处理“Thinking”后输出的自助力机制的值。

**ps：gpu可以对矩阵并行计算所有的输出，大大提速：自注意力机制的计算复杂度是O(n^2)，其中n是序列长度。尽管这听起来很高，但自注意力操作可以高度并行化，因为每个序列元素对的注意力计算是独立的；GPU擅长处理大规模并行任务，这使得自注意力机制能够在GPU上高效执行。由于自注意力涉及大量的矩阵乘法，这些操作可以在GPU上并行处理，从而加快计算速度 。**

#### Part3举个栗子

**第一步** ，计算出所有输入的查询向量，键向量，值向量。其生成的结果记为查询矩阵Q，键矩阵K，值矩阵V。

计算过程如下图所示。矩阵中的每一行都对应句子中的一个单词。我们仍可以注意到，词向量与查询向量和键向量和值向量的维度并不需要一致。

self-attention-matrix-calculation

**ps：在神经网络设置中，这些权重通常是很小的数字，一般会对这些权重矩阵使用适当的随机分布(例如高斯、Xavier和Kaiming分布)进行随机初始化。**

**接着** ，用下面的方程式便可计算出所有输入对应的输出。

self-attention-matrix-calculation-2

![](https://i-blog.csdnimg.cn/blog_migrate/cd229f741b1183ca820d4ea5727ad86d.png)

![图片](https://i-blog.csdnimg.cn/blog_migrate/404863933acbb890777f3ed9aac294bb.png)

![](https://i-blog.csdnimg.cn/blog_migrate/7866cd04ac7350ec00623433b53951e4.png)

![图片](https://i-blog.csdnimg.cn/blog_migrate/a7c276b0b0222e41821697a3e780afbc.png)

![](https://i-blog.csdnimg.cn/blog_migrate/ec4c839350b0de880ab50b7891e13fad.png)

![图片](https://i-blog.csdnimg.cn/blog_migrate/6cd812aa741533548bf66d97ea2fd8f7.png)

![](https://i-blog.csdnimg.cn/blog_migrate/2def09c0b29ccb0c3abbbbdd873b985b.png)

![图片](https://i-blog.csdnimg.cn/blog_migrate/26c9e964c9c66682eb8824c25ca0f3e2.png)

![](https://i-blog.csdnimg.cn/blog_migrate/d28c20cdf083618e112e756362a4d628.png)

![图片](https://i-blog.csdnimg.cn/blog_migrate/3fabea30afe362c864b261a7e4de1772.png)

计算所有输入的输出

**这里面有代码的例子：** [“AI”科普丨一文读懂自注意力机制：8大步骤图解+代码](https://mp.weixin.qq.com/s/hrAfZL_KEAJvEbSCG-zfDQ "“AI”科普丨一文读懂自注意力机制：8大步骤图解+代码")

---

### 3 Self-Attention的作用

那么，通过Self Attention有什么作用呢？这里仍然以机器翻译中的Self-Attention来说明，如下图是可视化地表示Self-Attention **在同一个英语句子内单词间产生的联系。**

![](https://i-blog.csdnimg.cn/blog_migrate/aa7b988f8fca8255a6af3a61c227d890.png)

从上图可以看出， **Self Attention可以捕获同一个句子中单词之间的一些句法特征或者语义特征** 。

同时，引入Self Attention后会 **更容易捕获句子中长距离的相互依赖的特征** ，因为如果是RNN或者LSTM，需要依次序序列计算，对于远距离的相互依赖的特征，要经过若干时间步步骤的信息累积才能将两者联系起来，而距离越远，有效捕获的可能性越小。

但是 **Self Attention在计算过程中会直接将句子中任意两个单词的联系通过一个计算步骤直接联系起来，所以远距离依赖特征之间的距离被极大缩短，有利于有效地利用这些特征** 。除此外， **Self Attention对于增加计算的并行性也有直接帮助作用** 。这是为何Self Attention逐渐被广泛使用的主要原因。

---

### 4 Self-Attention的问题

Self-Attention **虽然考虑了所有的输入向量，但没有考虑到向量的位置信息** 。 **有学者提出可以通过位置编码(Positional Encoding)来解决这个问题，就是把位置信息添加到输入序列中，让输入数据本身就带有位置信息，该方法我们将在后续的Transformer中详细介绍。**

![](https://i-blog.csdnimg.cn/direct/efcd943af2c747a8bcb5e73c56183851.png)

![](https://i-blog.csdnimg.cn/direct/ec46a1ebc2574c4b88a6f2529597c9ff.png)

**自注意力存在的问题主要有：没有考虑位置信息；自注意力计算结果互斥；计算速度慢；需要充足的数据量等**

**解决方案：融入位置编码（Transformer）；引入多头自注意力机制（Transformer）；**

### 5 关于多头注意力机制

**多头注意力(Multi-Head Attention)**

因为一段文字可能蕴含了比如情感维度、时间维度、逻辑维度等很多维度的特征，为了能从不同的维度抓住输入信息的重点，chatGPT使用了多头注意力机制(multi-head attention)。

而所谓多头注意力，简单说就是把输入序列投影为多组不同的Query，Key，Value，并行分别计算后，再把各组计算的结果合并作为最终的结果，通过使用多头注意力机制，ChatGPT可以更好地捕获来自输入的多维度特征，提高模型的表达能力和泛化能力，并减少过拟合的风险。

**多头注意力机制的目的是为了从多个维度捕捉提取更多的特征，从多个“头”得到不同的Self-Attention Score，提高模型表现。**

**首先放一张论文原文中的多头注意力机制的架构（Multi-Head Attention），可以看到（V,K,Q）三个矩阵通过h个线性变换（Linear），分别得到h组（V,K,Q）矩阵，每一组（V,K,Q）经过Attention计算，得到h个Attention Score并进行拼接（Concat），最后通过一个线性变换得到输出，其维度与输入词向量的维度一致，其中h就是多头注意力机制的“头数”。**

![图片](https://i-blog.csdnimg.cn/blog_migrate/5fd02f549e1d4e0dc33f8ef36202011d.png)

**下图为更直观的表示论文中的计算过程** ，以输入词“ **X** =\[‘图’, ’书’, ’馆’\]”为例，句子长度为3，词向量的维度为4，这里将词向量分为2个头， 线性 变换后得到2组（V0,K0,Q0）和（V1,K1,Q1），每组（V,K,Q）进行Self-Attention计算得到两个Score即（Z0和Z1），将Z0和Z1进行拼接Concat后进行线性变换得到输出向量Z，其维度与输入矩阵维度相同。

**ps：多头注意力机制代码实现和论文里的模式不一样哦（详见下面）！！！**

**下图是代码实现的过程，不同于论文，代码中对（V,K,Q）进行一次线性变换，然后在特征维度上进行h次分割（在代码中就是通过矩阵转置transpose和维度变换torch.view）后得到h组（V,K,Q），分别计算Self-Attention Score后进行Concat拼接** （同样的通过一系列的transpose和torch.view），最后通过线性变换得到最后的输出。

最后附一张代码截图，馆长的代码可能写的不太规范和严谨 **，仅根据论文实现主要功能** ， **并没有体现Dropout和mask等全面的功能，主要是为了通过代码实现来更好的理解Transformer中多头注意力机制，** 仅供学习交流参考，在实战中的代码需要进一步完善。

**ps：多头注意力中K、Q、V的线性层具有相同输入和输出尺寸是一种常见且实用的设计选择**

![](https://i-blog.csdnimg.cn/direct/426274cc093343c8b6300f24c2aa766e.png)

![](https://i-blog.csdnimg.cn/direct/66b35b9a1d2d4e3d88ccbb91d2642c69.png)

![图片](https://i-blog.csdnimg.cn/blog_migrate/721e994cc968af0dd7b5c608bb9760df.png)

```python
import torch

import torch.nn as nn

import torch.nn.functional as F

 

# 这个代码中省略了输入矩阵x转变为qkv的过程！！！

 

class MultiHeadAttention(nn.Module):

    def __init__(self, heads, d_model):

        super(MultiHeadAttention, self).__init__()

        self.d_model = d_model

        self.heads = heads

        # 定义K, Q, V的权重矩阵

        # 多头注意力中K、Q、V的线性层具有相同输入和输出尺寸是一种常见且实用的设计选择！！！

        self.k_linear = nn.Linear(d_model, d_model)

        self.q_linear = nn.Linear(d_model, d_model)

        self.v_linear = nn.Linear(d_model, d_model)

        # 分头后的维度

        self.d_token = d_model // heads

        # 定义输出权重矩阵

        self.out = nn.Linear(d_model, d_model)

        

    def forward(self, q, k, v):

        # 计算batch大小

        batch = q.size(0)

        

        # 线性变换后的Q, K, V，然后分割成多个头

        k = self.k_linear(k).view(batch, -1, self.heads, self.d_token)

        q = self.q_linear(q).view(batch, -1, self.heads, self.d_token)

        v = self.v_linear(v).view(batch, -1, self.heads, self.d_token)

        

        # 转置调整维度，以计算注意力分数

        k = k.transpose(1, 2)  # 形状变为 [batch, heads, seq_len, d_token]

2

2

        

        # 计算自注意力分数

        scores = self.attention(q, k, v, self.d_token)

        

        # 调整形状以进行拼接

        scores = scores.transpose(1, 2).contiguous().view(batch, -1, self.d_model)

        

        # 通过输出权重矩阵进行线性变换

        output = self.out(scores)

        return output

 

    @staticmethod

    def attention(q, k, v, d_token):

        # 计算注意力分数 (q @ k^T) / sqrt(d_token)

1

        # 应用softmax归一化（沿着最后一个维度（dim=-1））

        attn = F.softmax(scores, dim=-1)

        # 计算加权的V

        output = torch.matmul(attn, v)

        return output
python运行
```

![](https://i-blog.csdnimg.cn/direct/8e4105cd1ca644d9abd00c839e869241.png)

代码解析：

k = self.k\_linear(k).view(batch, -1, self.heads, self.d\_token)  
q = self.q\_linear(q).view(batch, -1, self.heads, self.d\_token)  
v = self.v\_linear(v).view(batch, -1, self.heads, self.d\_token)

![](https://i-blog.csdnimg.cn/direct/549c51f25cb443f5910953d27fe6af02.png)

![](https://i-blog.csdnimg.cn/direct/941b88fb371e4d5d94240d39f5ce9624.png)

\# 转置调整维度，以计算注意力分数  
k = k.transpose(1, 2) # 形状变为 \[batch, heads, seq\_len, d\_token\]  
q = q.transpose(1, 2)  
v = v.transpose(1, 2)

![](https://i-blog.csdnimg.cn/direct/eb248ca6773841a0959ec8654c49d685.png)

scores.transpose(1, 2).contiguous().view(batch, -1, self.d\_model)

![](https://i-blog.csdnimg.cn/direct/a50ba1db5f87494c8510d5fc4ae29e2f.png)

![](https://i-blog.csdnimg.cn/direct/4fb1ef3aa6594f95950847a50bea486e.png)

scores = torch.matmul(q, k.transpose(-2, -1)) / torch.sqrt(d\_token)

![](https://i-blog.csdnimg.cn/direct/ee1c7511c52f455fa21f88c3d3200afa.png)

**ps：实际在Transformer中多头注意力机制：在计算出注意力结果并经过线性层和dropout之后，还需要和Q进行短接！！！** ![](https://i-blog.csdnimg.cn/direct/1a9e9bc243ae44a7afae8d8898376e44.png)

```python
# 多头注意力计算层

class MultiHead(torch.nn.Module):

    def __init__(self):

        super().__init__()

        self.fc_Q = torch.nn.Linear(32, 32)# 就是那三个初始化QKV的参数矩阵

32

32

 

32

 

True

 

        self.dropout = torch.nn.Dropout(p=0.1)

 

    def forward(self, Q, K, V, mask):

        # b句话,每句话50个词,每个词编码成32维向量

        # Q,K,V = [b, 50, 32]

        b = Q.shape[0]

 

        # 克隆Q：保留下原始的Q,后面要做短接用

        clone_Q = Q.clone() # 克隆副本，并不共享空间！

 

        # 规范化（论文是放在后面，但实际都是放在前面。经过广泛论证这样效果会更好，能更好地帮助模型收敛！）

        Q = self.norm(Q)

        K = self.norm(K)

        V = self.norm(V)

 

        # 线性运算,维度不变

        # [b, 50, 32] -> [b, 50, 32]

        K = self.fc_K(K)

        V = self.fc_V(V)

        Q = self.fc_Q(Q)

 

        # 拆分成多个头

        # b句话,每句话50个词,每个词编码成32维向量,4个头,每个头分到8维向量

        # [b, 50, 32] -> [b, 4, 50, 8]

        Q = Q.reshape(b, 50, 4, 8).permute(0, 2, 1, 3)

        K = K.reshape(b, 50, 4, 8).permute(0, 2, 1, 3)

        V = V.reshape(b, 50, 4, 8).permute(0, 2, 1, 3)

 

        # 计算注意力

        # [b, 4, 50, 8] -> [b, 50, 32]

        score = attention(Q, K, V, mask)

 

        # 计算输出,维度不变

        # [b, 50, 32] -> [b, 50, 32]

        score = self.dropout(self.out_fc(score))# dropout防止过拟合

 

        # 短接

        score = clone_Q + score

        return score
python运行
```

![](https://i-blog.csdnimg.cn/direct/9c51bd5896d64464b6df21d925e3ac00.png)