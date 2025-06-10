> ACL ARR 2025，港科大、出门问问

1. 现有 TTS 模型缺乏对语音属性的细粒度控制，且在 single-stream 方法中，语义和声学信息在 token 中纠缠在一起，导致无法独立地调整语音特征
2. 提出 Spark-TTS，基于 BiCodec 将语音分解为两种互补的 token：低比特率的语义 token 用于体现内容，固定长度的全局 token 用于说话人相关属性
3. 结合 Qwen2.5 LLM 和 CoT 实现了粗粒度属性控制（如性别、说话风格）和细粒度参数调整（如精确音高值、说话速率）
4. 提出 VoxBox，包含 10 万小时数据的数据集，包含全面的属性注释

## Introduction
1. 基于 codec 的 LLM 已成为 zero-shot TTS 的主流范式，合成的语音与人类语音几乎无法区分
2. 但是存在一些挑战：
    + 现有的 codec-based TTS 架构复杂，需要两个生成模型或者并行解码
    + 现有方法主要受限于参考音频合成，缺乏合成具有特定特征的语音的能力
    + 现有研究中使用的专有数据集使评估和方法比较变得困难
3. 提出 Spark-TTS，使用单个 codec LLM 实现 zero-shot TTS 和全面属性控制，同时保持与常规文本 LLM 的架构一致；提出 VoxBox 开源语音数据集；提出 BiCodec 作为 tokenization 框架，结合低比特率的语义 token 和固定长度的全局 token 捕捉语言内容和声学特征；在 BiCodec 基础上，利用 Qwen2.5 进行微调，将 TTS 集成到文本 LLM 中；实现分层属性系统，结合粗粒度标签（和细粒度数值，通过 CoT 预测框架进行协调

## 相关工作（略）

## BiCodec

BiCodec 将输入音频离散化为两种 token：
+ 语义 token（每秒 50 个 token），捕捉语言内容
+ 固定长度的全局 token，编码说话人属性和其他全局语音特征

### 架构


如图：
![](image/Pasted%20image%2020250604220512.png)

包含 Global Tokenizer 和 Semantic Tokenizer：
+ Global Tokenizer 从 Mel 频谱图中提取全局 token
+ Semantic Tokenizer 使用 wav2vec 2.0 的特征作为输入来提取语义 token

BiCodec 遵循 VQ-VAE 编解码器框架。解码器将离散 token 重构回音频。给定输入音频信号 $x \in [-1, 1]^T$，其中 $T$ 为采样数，BiCodec 工作如下：
$$
\begin{aligned}&\boldsymbol{z}=E_s(F(\boldsymbol{x})),\boldsymbol{g}=E_g(\mathbf{Mel}(\boldsymbol{x}))\\&\boldsymbol{g}_f=\text{CrossAttention}(\boldsymbol{g},\boldsymbol{h}),\\&\boldsymbol{z}_q=Q_s(\boldsymbol{z}),\boldsymbol{g}_q=Q_g(\boldsymbol{g}_f),\\&\boldsymbol{\hat{x}}=G(\boldsymbol{z}_q,A_g(\boldsymbol{g}_q)),\end{aligned}
$$
其中 $E_s(·)$ 是 semantic tokenizer 的编码器，$F(·)$ 是预训练的 wav2vec 2.0，$E_g(·)$ 是 global tokenizer 的编码器，$Mel(·)$ 用于从 $x$ 中提取 Mel 频谱图，$h$ 是与最终全局 token 序列长度匹配的可学习查询序列，$Q_s(·)$ 是使用 VQ 的量化层，$Q_g(·)$ 是使用 FSQ 的量化层，$A_g(·)$ 是具有池化层的聚合模块，$G(·)$ 是重构时域信号 $\hat{\boldsymbol{x}}$ 的解码器。

### 模型架构

semantic tokenizer 的编码器 $E_s$ 和解码器 $G$ 基于 ConvNeXt 块构建。对于 wav2vec 2.0（XLSR-53），选择第 11、14 和 16 层的特征进行平均以获得语义特征作为 semantic tokenizer 的输入。

global tokenizer 的编码器 $E_g$ 使用 ECAPA-TDNN 架构，在最终池化层之前使用 Wespeaker 的方法。编码后，global tokenizer 使用交叉注意力和一组可学习的查询来提取固定长度的 $g_f$。

semantic tokenizer 使用 single-codebook VQ 进行量化。global tokenizer 使用 FSQ。

### 训练

采用 GAN 方法最小化重构损失，使用 L1 特征匹配损失来优化 VQ 码本。频域重构损失为多尺度 Mel 频谱图的 L1 损失。使用多周期鉴别器和多带多尺度 STFT 鉴别器进行波形鉴别和频域鉴别。VQ 码本学习包括 codebook loss 和 commitment loss。
还使用了 wav2vec 2.0 重构损失来确保语义相关性。

## Spark-TTS

### 概览

如图：
![](image/Pasted%20image%2020250604222407.png)

Spark-TTS 采用 decoder-only transformer 架构，使用预训练的文本 LLM Qwen2.5-0.5B 作为骨干。Spark-TTS 直接采用 BiCodec 的解码器来生产音频。

除了 zero-shot TTS，还可以通过提供属性标签来创建语音。推理时，如果提供性别、音高和语速的属性标签，模型通过 CoT 预测精细的音高值、语速值、全局 token 和语义 token。如果没有提供属性标签，则从参考音频中提取全局 token，从而实现 zero-shot TTS。

### Tokenizer

文本 tokenizer 使用基于 BPE 的 tokenizer 处理原始文本，采用 Qwen2.5 的 tokenizer。

属性 tokenizer 在两个层次上编码属性信息：
+ 粗粒度：表示高级语音特征的属性标签，包括性别、音高（分为五个离散级别）和语速（分为五个离散级别）
+ 细粒度：使音高和语速能够精确控制的属性值，在 tokenization 时通过四舍五入到最近的整数进行量化

语音 tokenizer 由 global tokenizer 和 semantic tokenizer 组成。

### 训练

令 $T$ 表示 tokenized 的文本，$G$ 表示全局语音 token，zero-shot TTS 的优化目标定义如下：
$$\mathcal{L}_{zst}=-\sum_{t=1}^{T_o}\log P(o_t|\mathcal{T},\mathcal{G},\boldsymbol{o}_{<t};\theta_{LM})$$
其中 $o \in \mathbb{N}^{T}_{o}$ 表示要预测的 semantic token，$\theta_{LM}$ 表示 LM 的参数。

对于语音合成，优化目标定义为：
$$\mathcal{L}_{control}=-\sum_{t=1}^{T_c}\log P(c_t|\mathcal{T},\mathcal{A},\boldsymbol{c}_{<t};\theta_{LM})$$
其中 $\mathcal{A}$ 表示属性标签提示，输出 $\mathcal{C}$ 包含 $\mathcal{F}$、$\mathcal{G}$ 和 $\mathcal{S}$。其中 $\mathcal{F}$ 表示细粒度属性值提示，$\mathcal{S}$ 是语音 semantic token。
$L_{zst}$ 和 $L_{control}$ 混合训练，每个音频根据 $L_{zst}$ 和 $L_{control}$ 分别构建两个训练样本。

## VoxBox

### 概览

提出 VoxBox 数据集：
+ 所有数据源均来自开源数据集（收集了常见的 TTS 数据集和用于语音情感识别的数据集）
+ 每个音频文件标注性别、音高和语速
+ 清理文本质量较低的数据，清理后包含 470 万个音频文件，来自 29 个开源数据集，总计 102.5k 小时

### 清理和标注（略）

## 实验（略）
