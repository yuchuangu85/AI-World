---
title: "狗都能看懂的Self-Attention讲解_self attention-CSDN博客"
source: "https://blog.csdn.net/weixin_42392454/article/details/122478544"
author:
  - "[[成就一亿技术人!]]"
  - "[[hope_wisdom 发出的红包]]"
published:
created: 2026-04-12
description: "文章浏览阅读3w次，点赞161次，收藏473次。一文看懂self-attention_self attention"
tags:
  - "self-attention"
---
### 1、什么是 attention ？

在人 类 的理解中，对待问题是有明显的侧重。具体举个例子来说：“我喜欢踢足球，更喜欢打篮球。”，对于人类来说，显然知道这个人更喜欢打篮球。但对于深度学习来说，在不知道”更“这个字的含义前，是没办法知道这个结果的。所以在训练 模型 的时候，我们会加大“更”字的权重，让它 **在句子中的重要性** 获得更大的占比。比如：  

### 2、什么是self-attention？

在知道了attention在机器学习中的含义之后（下文都称之为注意力机制）。人为设计的注意力机制，是非常主观的，而且没有一个准则来评定，这个 权重 设置为多少才好。所以，如何让模型自己对变量的权重进行 **自赋值** 成了一个问题，这个 **权重自赋值** 的过程也就是self-attention。

![self-attention](https://i-blog.csdnimg.cn/blog_migrate/5c8492ac2d58f9e58248e4fba43fbcba.png#pic_center)

### 3、self-attention的原理

定义：假设有四个输入变量，，，，希望它们经过一个self-attention layer之后变为，，，。

拿和做例子，这个结果是综合了，，，而得出来的一个结果。

既然得到一个是要综合所有的才行，那么最直接的做法就是与，，都做一次运算，得到的结果就代表了这个变量的注意力系数。直接做乘法太暴力了，所以选择一个更柔和的方法：引入三个变量，，，这三个变量与相乘得到，，，同样的方法对，，都做一次。

至于这里的具体代表什么，下面就慢慢展开讲解。

![qkv](https://i-blog.csdnimg.cn/blog_migrate/94df43e0272ded4979547a6349555985.png#pic_center)

那么拿自己的与别人的相乘就可以得到一个系数。这里在和其他的做内积时，可近似的看成是在做相似度计算。比如：  

在实际的神经网络计算过程中，还得除于一个缩放系数，这个是指和的维度，因为和会做内积，所以维度是一样的。之所以要除，是因为做完内积之后，会随着它们的维度增大而增大，除相当于标准化。

![alpha](https://i-blog.csdnimg.cn/blog_migrate/91d6e4e58b1d4c42db6c60bfdc3af44d.png#pic_center)

得到了四个之后，我们分别对其进行softmax，得到四个，增加模型的非线性。

![softmax](https://i-blog.csdnimg.cn/blog_migrate/8898fb0100510dcb2e78a8d3c151471f.png#pic_center)

四个分别是，，，，别忘了还有我们一开始计算出来的，，，。可能有读者就会问了，直接把各个直接与各个相乘不就得出了最后的结果了吗？虽然这么说也没错，但为了增加网络深度，将变成也可以减少原始的对最终注意力计算的影响。

那么距离最后计算出只剩最后一步，我们将所有的与所有的分别相乘，然后求和，就得出啦！具体计算如下：  
  
公式简化为：  
  
![sum](https://i-blog.csdnimg.cn/blog_migrate/91a5baae9e6804aa10a5e5fb2d1f3201.png#pic_center)

同样的计算过程，我们对剩下的都进行一次，就可以得到，，。每个都是综合了 **每个之间的相关性** 计算出来的，这个相关性就是我们所说的 **注意力机制**,。那么我们将这样的计算层称为self-attention layer。

![self-attention layer](https://i-blog.csdnimg.cn/blog_migrate/0cc098509f6114d5dbd82dd9562e0710.png#pic_center)

我们把一个句子中的每个字代入上图的，，，，就可以将self-attention应用到自然语言处理的领域了。

### 4、self-attention的优点

从第三节\[self-attention的原理\](## 3、self-attention的原理)中可以看出，这一层需要学习的参数只有，，，大部分变量来自于内部计算得出来的，所以它的参数量少但每个参数所涵盖的信息多，这是它的第一个优点。

每个的计算都是独立的，这一点相比之前的RNN来说很不一样，RNN是需要等前面的算完了才能算，是串行的。所以RNN无论是训练还是推理，都会因为不能计算并行而变慢，这是它的的第二个优点。

RNN的一个最大的问题是：前面的变量在经过多次RNN计算后，已经失去了原有的特征。越到后面，最前面的变量占比就越小，这是一个很反人类的设计。而self-attention在每次计算中都能保证每个输入变量的初始占比是一样的，这样才能保证经过self-attention layer计算后他的注意力系数是可信的。

所以总结下来，它的三个优点分别是：

- 需要学习的参数量少
- 可以并行计算
- 能够保证每个变量初始占比是一样的

### 5、Multi-head self-attention

这里继续讲解multi-head self-attention，所谓head也就是指一个衍生出几个。上述所讲解的self-attention是基于single-head的。以2 head为例：

首先，先生成，，。然后，接下来就和single-head不一样了，生成，生成的方式有两种：

1.乘上一个得到，乘上得到，这个和single-head的生成是差不多的；  
2.直接从通道维，平均拆分成两个，得到；

这两种方式，在最后结果上都差不多。至于为啥，后面会讲一下原因。

那么这里的图解使用第1个方式，先得到，，。对做同样的操作得到，，。这边需要注意的一点，是要和做矩阵乘法，而非，一一对应。后面计算就和single-head一样了，最后得到。

![multi-head_1](https://i-blog.csdnimg.cn/blog_migrate/db253eed6765faa21d6e76e4e831be87.png#pic_center)

第二步，对，，做一样的操作，得到。

![multi-head_2](https://i-blog.csdnimg.cn/blog_migrate/294650336afa3a19416811c2f3fc4f28.png#pic_center)

这里我们算出的，是同维度的，我们可以将其concat在一起，再通过一个把他转成想要的维度。这也就不难理解，为什么说multi-head的两种生成方式是一样的，因为最终决定是输出维度的是。我们可以将multi-head的过程看成是cnn中的隐藏层，multi-head的数量也就对应着Conv2D的filter数量，每一个head各司其职，提取不同的特征。

![multi-head_3](https://i-blog.csdnimg.cn/blog_migrate/b03301b1cc701f938bfd88cbb8863e7c.png#pic_center)

### 6、Position Encoding

最后需要讲解的一点是位置编码。如果读者已经理解了self-attention的原理，不难发现，对于各个来说，无论相隔多远多近，互相影响程度是一样的，最粗暴的的情况当然是两个离的越远，互相之间的attention越小，当然一些倒装句的存更需要position encoding。例如：“我爱你”和“你爱我”输入到self-attention layer，其计算的结果是一样的。

![position encoding](https://i-blog.csdnimg.cn/blog_migrate/acc7167e037a01189ab0178a5e6f5cb4.png#pic_center)

这里我们采用的位置编码是和一样维度的，融合方式是：直接与相加。这里的内容是人为设定的，可以被训练，也有固定的，即sin cosine function。作者尝试后，说二者效果差不多。

还有一个延申的问题： **为什么融合方式是相加，而非concat？**

一种从线性代数的角度的说法是：

假设位置编码输入是：是位置编码的原始输入，它是类似one-hot的向量concat而来，将和concat在一起之后，乘上一个，我们可以将看成两个部分，和。整个相乘的过程可以拆成，可视作是，可视作。所以直接相加和concat其实没什么不同。

![position encoding add](https://i-blog.csdnimg.cn/blog_migrate/ede38337486ca981ee4284f074784b53.png#pic_center)

### 7、self-attention的变种和应用

~~本文所说的self-attention是single-head的，也就是每个只对应一个，还有multi-head的，即对应多个。而原理也是基于NLP方向讲解的，后续也有人将它应用在了CV中，就是大名鼎鼎的Vision Transformer。这些就以后有空再更新吧。~~

另外附上一个讲的也很好的博客： [超详细图解Self-Attention](https://zhuanlan.zhihu.com/p/410776234) 。

### 8、总结

至此self-attention的相关内容以及全部讲解完了，对CV部分有兴趣的同学可以继续阅读 [Vision Transformer](https://blog.csdn.net/weixin_42392454/article/details/122667271) 的讲解。