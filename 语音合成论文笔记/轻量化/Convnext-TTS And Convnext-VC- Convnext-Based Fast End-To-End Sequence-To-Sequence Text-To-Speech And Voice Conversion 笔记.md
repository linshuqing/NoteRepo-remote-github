> ICASSP 2024，NII、名古屋大学

1. 提出 Transformer-free 的基于 ConvNeXt 的 encoder 和 decoder，提高 E2E-S2S TTS 和 VC 模型的合成质量并提高推理速度
2. 提出 ConvNeXt-TTS 和 ConvNeXt-VC，包括 WaveNeXt vocoder
3. 在 Hi-Fi-CAPTAIN 上的实验表明 ConvNeXt-based encoder 和 decoder 的推理速度比 Transformer-based encoder 和 decoder 快三倍，同时提高了合成质量

> 结构和 JETS 类似，但是把其中的 Fastspeech 中的 Transformer 替换成 ConvNext，把 HiFi-GAN 替换为 WaveNext

## Introduction

1. Transformer-based encoders and decoders  是现有 S2S-TTS 和 S2S-VC 的标准
2. ConvNeXt 可以替代 Transformer，提高识别准确率和推理速度
3. 提出基于 ConvNeXt 的 E2E-S2S TTS 和 VC 模型：
    1. 第一次将 ConvNeXt 架构用于 TTS 和 VC 框架的 encoder-decoder 模型
    2. 提出 ConvNeXt-TTS 和 ConvNeXt-VC，结合 WaveNeXt neural vocoder 来进一步提高推理速度
4. 在 Hi-Fi-CAPTAIN 上的实验表明 ConvNeXt-based encoder 和 decoder 的推理速度比 Transformer-based encoder 和 decoder 快三倍，同时提高了合成质量

## 传统方法

模型结构对比图：
![](image/Pasted%20image%2020250215111842.png)

### JETS 和 JETS-VC (Transformer + HiFi-GAN)

[JETS- Jointly Training FastSpeech2 and HiFi-GAN for End to End Text to Speech 笔记](JETS-%20Jointly%20Training%20FastSpeech2%20and%20HiFi-GAN%20for%20End%20to%20End%20Text%20to%20Speech%20笔记.md) 联合训练 FastSpeech-2-based AM 和 HiFi-GAN-based neural vocoder（见上图 a），没有中间的 mel-spectrograms 和外部的 aligners。

### WaveNeXt neural vocoder

vocoder 结构对比如下：
![](image/Pasted%20image%2020250215111028.png)

[Vocos- Closing the gap between time-domain and Fourier-based neural vocoders for high-quality audio synthesis 笔记](Vocos-%20Closing%20the%20gap%20between%20time-domain%20and%20Fourier-based%20neural%20vocoders%20for%20high-quality%20audio%20synthesis%20笔记.md) 基于 ConvNeXt 模块从 mel-spectrograms 预测 STFT 谱，然后直接转换为 speech waveforms，但是合成质量不如 HiFi-GAN。

[WaveNeXt- ConvNeXt-Based Fast Neural Vocoder Without ISTFT layer](WaveNeXt-%20ConvNeXt-Based%20Fast%20Neural%20Vocoder%20Without%20ISTFT%20layer.md) 用可训练的线性层替代 STFT-based upsampling layer，可以在单核 CPU 上实现 0.04 的 RTF，比 Vocos 和 HiFi-GAN V2 合成质量更高。

### JETS-WN 和 JETS-WN-VC (Transformer + WaveNeXt)

JEETS-WN 和 JETS-WN-VC （上图 b）用 FastSpeech 2-based AM 和 Vocos 的 discriminator 训练，其损失函数定义为：
$$\mathcal{L}_{G,JETS-WN}=\mathcal{L}_G+w_{var}\mathcal{L}_{var}+w_{align}\mathcal{L}_{align}$$
且 $\mathcal{L}_{D,JETS-WN}=\mathcal{L}_D$，其中 $\mathcal{L}_G$ 和 $\mathcal{L}_D$ 分别为 Vocos 的 generator 和 discriminator 损失函数，$\mathcal{L}_{var}$ 和 $\mathcal{L}_{align}$ 分别为 JETS 的 variance loss 和 alignment loss，$w_{var}$ 和 $w_{align}$ 分别为 $\mathcal{L}_{var}$ 和 $\mathcal{L}_{align}$ 的权重系数。JETS-WN 和 WaveNeXt 一样，可以比 JETS-Vocos 实现更高的合成质量。

## 提出的方法

### CN-JETS 和 CN-JETS-VC (ConvNeXt + HiFi-GAN)

ConvNeXt 可以提高图像识别的准确性和推理速度，虽然不使用 Transformer，但是其行为类似 Swin Transformer。ConvNeXt 由 layer normalization layers、depthwise convolution layers、pointwise convolution layers 和 GELU 构成。depthwise convolution 对应于 Transformer 中的 self-attention 的加权和。ConvNeXt 可以比 Swin Transformer 在图像识别中实现更快的推理速度和更高的准确性。

于是提出了两个基于 ConvNeXt 的 E2E-S2S TTS 和 VC 模型，CN-JETS 和 CN-JETS-VC（如上图 c）。使用 ConvNeXt-based encoder 和 decoder 替代 JETS 和 JETS-VC 中的 Transformer-based encoder 和 decoder：
+ 使用 Vocos 和 WaveNeXt 中的 ConvNeXt blocks 和 1D convolutions
+ 使用 MAS 进行对齐训练
+ 使用 HiFi-GAN 中的 discriminator

### ConvNeXt-TTS 和 ConvNeXt-VC (ConvNeXt + WaveNeXt)

为了进一步提高推理速度，提出 ConvNeXt-TTS 和 ConvNeXt-VC，将 WaveNeXt neural vocoder 引入 CN-JETS 和 CN-JETS-VC 中，替代 HiFi-GAN（如上图 d）。和 JETS-WN 和 JETS-WN-VC 一样，使用 Vocos 中的 discriminator loss functions。

## 实验

数据集：Hi-Fi-CAPTAIN，输入 acoustic features 为 80 维 mel-spectrograms，STFT 长度和 shift 长度分别为 1024 和 256

模型设置：见原论文

评估标准：MCD、logfo RMSE 和 CER 作为客观评价标准，MOS 作为主观评价标准

客观指标如下：
![](image/Pasted%20image%2020250215113443.png)

实验结果：
+ FastSpeech-2-based AM、ConvNeXt-based AM、HiFi-GAN 和 WaveNeXt-based neural vocoders 的 RTFs 分别为 0.03、0.01、0.80 和 0.04
+ ConvNeXt-based encoder 和 decoder 的推理速度比 Transformer-based encoder 和 decoder 快三倍，同时提高了合成质量
+ ConvNeXt-TTS 和 ConvNeXt-VC 在单核 CPU 上实现 0.05 的 RTF
+ ConvNeXt-TTS 和 ConvNeXt-VC 在 ASR 准确率和 MCD 上表现最好，同时在合成质量和说话人相似度上显著优于 JETS-WN 和 JETS-WN-VC
+ CN-JETS 和 CN-JETS-VC 在合成质量和说话人相似度上优于 JETS 和 JETS-VC
