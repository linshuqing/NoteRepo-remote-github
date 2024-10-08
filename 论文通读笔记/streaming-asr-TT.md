<!--
 * @Description: 
 * @Autor: 郭印林
 * @Date: 2022-08-10 13:46:04
 * @LastEditors: 郭印林
 * @LastEditTime: 2022-08-11 14:03:20
-->
## Developing real-time streaming transformer transducer for speech recognition on large-scale dataset 笔记

1. 结合 Transformer-XL and chunk-wise 处理方法，设计 Streamable Transformer Transducer（T-T）


### Introduction
1. 现有流式方法及其缺点：
    + Time-restricted 法，对左边和右边的context进行mask，缺点是层数的增加引入很大的延迟
    + chunk-wise 法：把输入语音分成几个 chunks，每个 chunks 分别做识别，，缺点是精度下降
    + Memory based 法：引入上下文向量编码历史信息，同时结合chunk-wise方法减少运行时间，缺点是破坏了Transformer的并行性，训练时间更长了
2. 本文目标是实现  streaming Transformer and Conformer Transducer 模型，在训练时间、推理时间和精度上达到平衡。
    + 结合 Transformer-XL and chunk-wise 处理方法
    + chunks 之间没有 overlap
    + 在 360ms 的look ahead 情况下，CPU上的 RTF 为 0.25

### 模型细节
1. 采取的还是RNN-T的结果，只不过是 transformer 为 encoder，LSTM 为 predictor
2. 已有实验证明，相对位置嵌入效果比绝对位置嵌入好
3. 采用了Conformer中的结构，把 depth-wise CNN 变成 causal depth-wise CNN 来减少延迟
4. 采用 truncated history 和 limited future information 来平衡计算效率和精度，缺点就是层数的增加线性增加了延迟。
5. 采用mask方法，首先分成多个chunks，然后根据以下规则构造mask矩阵：
    + 同一个chunks里面的帧可以互相看见
    > 每个chunks中靠前的帧可以看到这个chunks中未来的帧，靠后的能看到的未来帧就越来越少
    + 不同chunks的两个帧，左边的不能看到右边的
    + 不同chunks的两个帧，通过两帧的距离小于历史窗口大小，那么右边的可以看到左边的

    图例和mask矩阵如下：
    ![1660113979882](image/streaming-asr-TT/1660113979882.png)
    好处是，左边的上下文可以（随层）线性增加，而禁止右边的感受野扩大（蓝色的为对输入的感受野）。
6. 推理优化：
    + Caching： Attention 计算中的 K 和 V 是可以缓存的
    + Chunk-wise 计算：采用矩阵乘法同时计算多个attention输出
