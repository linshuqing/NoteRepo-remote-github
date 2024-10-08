
1. 发现静音段持续时间的不均匀分布和语音的真假有关
2. 相比于虚假语言，真实语音的前后静音段时间更长
3. 作者通过实验表明，仅在静音段上训练模型都可以得到很高的准确率，因此之前的工作可能在某种程度上通过依赖于静音段持续时间而无意中学会了区分真实和虚假语音

## Introduction

1. 本文关注 LA 检测
2. 作者注意到训练数据集中静音段分布的不规则性，从而模型可能会部分根据静音时间来做出决定
3. 神经网络是数据驱动的方法，很可能过拟合这种不规则分布
4. 本文主要回答以下问题：以前的方法是否已经学会了仅仅或至少部分地基于前后静音段持续时间来区分音频

## LA 数据集描述

论文中给出了 2019 LA数据集中 train、dev、eval 数据集不同类别的持续时间，发现这个时间似乎是一个有用的区分真假的特征：相比于虚假语音，真实语音的静音段时间更长，因为攻击是由文本转语音（TTS）系统创建的，通常设计为输出中没有前后静音。

## 模型描述

实现了一个简单的 FCNN。

实现了几个模型，ResNet、LSTM 、 CNN-GRU 和 RawNet2。

## 实验设计和结果

训练参数见论文。

训练了前面提到的 FCNN，但是只使用静音段时间（单位s）作为训练特征，如果静音段时间确实是一个特征，那么最终的 EER 应该小于 50%（因为随机猜测就是 50%），结果如下：![[Pasted image 20221214100031.png]]
有理由怀疑模型学习了静音段时间来区分真实和虚假语音。

同时使用 CQT 特征和前面提到的三个模型，分别在有无静音段的情况下来训练模型，使用了两种预处理技术来实现：
+ Time-wise subselection：就是随机选一个片段。。
+ Trim Silence：就是去除静音段
一共12个模型，结果如下：![[Pasted image 20221214100500.png]]
去掉静音段之后，性能发生大幅下降。这说明，目标信息大量包含在静音段中。

同时也在 RawNet2 上去除静音段来观察性能，结果如图：![[Pasted image 20221214100811.png]]

最后尝试将静音段的持续时间作为特征提供给模型，但是并没有使EER有改善，作者给的解释是，模型能够很容易地自行提取静音段时间，因此额外提供时间是没有好处的。

## 讨论  

静音段通常具有（不可察觉的）属性和特征，包括微弱的背景噪声、所用麦克风的伪影等。

合理的假设是，TTS系统不能完美地伪装静音，因此静音可以合法地用于将音频分类为欺骗/真实。

通常情况下，允许ASVspoof等数据集中的记录包括前导和尾随静音。然而，问题是时间的分布非常不均匀。

在这种情况下，模型学习分类不是由于静音本身的性质和特征，而是由于其持续时间。因此，如果要在数据集中包含静音段，则应在真实和虚假样本之间均匀分布。

