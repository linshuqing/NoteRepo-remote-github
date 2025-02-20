FastDiff> ICASSP 2023，University of the Philippines（菲律宾大学），一个作者

1. 提出 EfficientSpeech，可以实时地在 ARM CPU 上合成波形，采用浅层的、非自回归的、金字塔结构的 Transformer，形成一个 U-Net
2. 266k 参数，相比于 FastSpeech 2 只有轻微的性能下降

> Encoder + Adaptor + Decoder 的架构，大致和 FastSpeech 差不多，但是每个模块都单独轻量化设计了。

## Introduction

1. 提出 EfficientSpeech，可以用于边缘设备的 TTS 模型，包含：
	1. 浅层的 U-Net 金字塔 transformer 作为 phoneme encoder
	2. 浅层的转置卷积作为 mel 谱 decoder
2. 只有 266k 的参数，如果包含 HiFiGAN vocoder，总参数约为 1.2M，可以在单个 GPU 上训练 12 小时

## 模型架构

![](image/Pasted%20image%2020240106102258.png)

如图，phoneme 序列 $\boldsymbol{x}_{phone}\in\mathbb{R}^{N\times d}$ 为 phoneme embedding，所有的卷积都是 1D 的，$N$ 为 phoneme 长度，$d$ 为 embedding 维度。

Phoneme Encoder 包含两个 transformer 模块，每个都包含 depth-wise separable convolution、Self-Attention 和 Mix-FFN。

第一个 transformer 模块保留序列长度，但是减少 1/4 的特征维度，第二个 transformer 模块对序列长度减半，但是加倍特征维度。最后通过 feature fuser 和其中的上采样层，得到最终的 phoneme 特征序列，维度为 $N\times \frac{d}{4}$。

Acoustic Features and Decoders 模块用的是 FastSpeech 2 中的方法，迫使网络预测能量 $\boldsymbol{y}_e$，pitch $\boldsymbol{y}_p$，duration $\boldsymbol{y}_d$，而且这里用的是并行的输出。

然后再进行特征融合和上采样，最终得到 $M\times d$ 的特征。

最后 Mel Spectrogram Decoder 包含两层线性层+两层的 depth-wise sep-arable convolution，用的是 Tanh 激活 + LN。

### 模型训练

数据集：LJSpeech

g2p 产生 phoneme，MFA 来获得 duration。pitch 通过 STFT 得到，energy 通过 WORLD 得到。

总损失如下：
$$\mathcal{L}=\alpha\mathcal{L}_{mel}+\beta\mathcal{L}_p+\gamma\mathcal{L}_e+\lambda\mathcal{L}_d$$

> 不涉及到 GAN。

## 实验结果（略）