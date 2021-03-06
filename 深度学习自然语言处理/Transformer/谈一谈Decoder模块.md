### 谈一谈Decoder模块

本文主要是谈一些比较容易误解的细节点，说实话，把自己的理解用文字表达出来真是个细致活。

如果觉得对您有点帮助，帮忙点个在看或者赞。

#### 一个小小的问题

我先说一个自己花了点时间才琢磨出来的东西，其实不难，就是当时没转过弯来。

我们都知道，decoder的交互层，Q矩阵来自本身，K/V矩阵来自整个Encoder层输出。

但是对于每个单词都会有一个encoder的输出，那么K/V矩阵是用的其中哪个输出计算过来的？

我这个问题的问法其实是错误的。

我当时的理解背景是认为这个交互的过程很类似seq2seq的attention，它一般是使用最后一个时刻的隐层输出作为context vector。

我基于此产生了上面这个问题，这个K/V矩阵应该由哪个位置单词（对比RNN就是哪个时刻的单词）的输出生成。

后来看了一下代码，才明白自己错在哪里？

K/V矩阵的计算不是来自于某一个单词的输出，而是所有单词的输出汇总计算K/V矩阵。这个过程和在Encoder中计算K/V矩阵是一样的，只不过放在了交互层，一时没想明白。

#### 正文

与Encoder很类似，Decoder同样由完全相同的N个大模块堆叠而成，原论文中N为6。

每个大的模块分为三部分：多头注意力层，交互层，前馈神经层；每个层内部尾端都含有 Add&Norm。

和Encoder重复的内容我就跳过了，之前讲过，没看过的同学可以去看那个文章。

##### 多头自注意力层

首先谈一下多头自注意力层，这里需要注意的细节点是，需要对当前单词和之后的单词做mask。

为什么需要mask？

最常规的解释就是在预测阶段，你的模型看不见当前时刻的输出以及未来时刻单词。

这句话其实有点绕，如果读的不仔细会让人误解为mask的时候需要把当前时刻的单词也mask掉...(拖出去斩了吧)。

从代码角度讲，你只需要把当前时刻之后所有单词mask掉就好了。

我自己对这句话的理解是我们需要确保模型在训练和测试的时候没有GAP。

举个简单的例子来理解，如果做机器翻译，你需要翻译出来的句子是 "我/爱/吃/苹果"。

当前时刻是”爱“这个单词作为输入的一部分，另一部分是上一个时刻”我“作为输入的时候的输出值。

当然在机器翻译中，我们一般使用 teacher forcing加速收敛，所以这里就使用”我“作为当前时刻输入的另一个部分。

所以这个时候，输入就是”我“的编码信息和”爱“的编码信息（当然还有位置编码）。

我要预测的是”吃“这个单词。

如果我们没有mask，模型也是可以运行的，也就说此时”吃“和”苹果“两个词对”爱“这个时刻的输出是有贡献的。

那么问题来了，测试数据中你根本没有ground truth，你怎么办？

也就说，训练的时候，你的模型是基于知道这个时刻后面的单词进行的训练，但是测试的时候，做机器翻译，你不知道自己应该翻译出来什么东西。

这就是问题的核心。

你训练模型的时候，一部分精力花在了”吃“和”苹果“两个词上，这不就是无用功吗？

所以，确保模型在训练和测试的时候没有GAP，我们需要mask掉”吃“和”苹果“两个词。

##### 交互模块

这一块需要注意的就是之前文章提到的，Q矩阵来自本身，K/V矩阵来自encoder的输出。

还有一个细节点是，K/V矩阵对应的是来自整个encoder的输出。

如果看transformer那个经典图的话，初期很容易理解为encoder和decode对应的每一层互相交互，这是不对的。

是整个输出与decoder做交互。