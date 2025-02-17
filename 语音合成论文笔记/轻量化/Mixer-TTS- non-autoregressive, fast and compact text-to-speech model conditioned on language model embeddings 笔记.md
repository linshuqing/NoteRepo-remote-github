> ICASSP 2022，NVIDIA

1. 提出 Mixer-TTS，用于 mel 谱生成的非自回归模型
2. 基于 MLP-Mixer 结构，基本模型包含 pitch 和 duration predictors，后者采用 无监督的 speech-to-text 框架获得；拓展模型还使用了从预训练的语言模型中得到的 token embedding

## Introduction

1. 提出 Mixer-TTS，基于 MLP-Mixer 结构，基本模型和 FastPitch 类似，拓展版采用预训练的 LM 来引入韵律，提高发音质量
2. 接上 HiFi-GAN 后，在 LJSpeech 上的 MOS 为 4.05（GT 为4.27）
3. 基本版 19.2M 参数，拓展版 24M 参数

## 模型架构

![](image/Pasted%20image%2020240117110327.png)

先编码文本然后通过 GT duration 对齐到音频特征，然后计算 phoneme-level pitch，然后送到 length regulator 中拓展，最后 decoder 生成 mel 谱。

基本的结构和 [FastPitch- Parallel Text-to-speech with Pitch Prediction 笔记](FastPitch-%20Parallel%20Text-to-speech%20with%20Pitch%20Prediction%20笔记.md) 很类似，两个不同：
+ 把所有的 FFN 层替换为 Mixer-TTS blocks
+ 用无监督的 speech-to-text 对齐框架训练 duration predictor
+ 拓展版还包含一个 pretrained LM embedding

训练的损失包含，aligner loss 和 mel 谱、duration、pitch 三者和对应的 GT 之间的 MSE loss：
$$L=L_{aligner}+L_{mel}+0.1\cdot L_{durs}+0.1\cdot L_{pitch}$$

### Mixer-TTS Block

MLP-Mixer 来自 CV，基于互斥的 MLP，对输入做两个关键操作：
+ 混合 per-location features
+ 混合 spatial information

两个操作通过级连的两层 MLP 实现，第一个 MLP 通过所谓的 expansion factor 增加 channel 数，第二个 MLP 减少到原来的 channel。但是这个只适用于输入维度是固定的，而 TTS 的输入是动态变化的，于是将 MLP 替换为 depth-wise 1D 卷积进行所谓的 time-mixing，其他不变，如图：
![](image/Pasted%20image%2020240117112608.png)

encoder 包含 6 个级连的 Mixer-TTS 模块，decoder 包含 9 个。

### Speech-to-text 对齐框架

通过采用无监督的对齐方法 [One TTS Alignment To Rule Them All 笔记](../对齐/One%20TTS%20Alignment%20To%20Rule%20Them%20All%20笔记.md) 联合训练一个 speech-to-text 对齐框架。
> 注意这里 speech-to-text 不是语音转文本，而是 语音-文本 之间的对齐。

### 拓展版 Mixer-TTS

采用 HuggingFace 中预训练的 ALBERT 模型，直接用它提取固定好的 embedding，由于用的是不同的 tokenizer，原始的 tokenized text 和 LM 中的不一样，于是采用 head 为 1  的 self-attention 模块，把原始的 text encoder 的 embedding $t_e$ 作为 Q，LM 中的 $lm\_t_{e}$ 作为 K V 进行特征混合。

## 结果（略）