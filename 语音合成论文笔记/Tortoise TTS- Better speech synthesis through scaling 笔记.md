> arxiv 2023 未发表，OpenAI 研究员 James Betker

1. 提出 TorToise，将图像生成的 autoregressive transformers 和 DDPMs 方法用到语音合成，支持多音色
2. 完全开源：https://github.com/neonbjb/tortoise-tts

> 过程完全参考 DALL-E，采用 LM 得到 codes，然后用 diffusion 从 codes 得到 mel 谱，然后用 vocoder 合成音频。引入 CLVP 进行 re-ranking。

##  背景（略）

## 方法

### 联合 自回归 decoder 和 DDPMs

首先：
+ 自回归模型擅长在视觉、文本和语音等不对齐的 domain 之间转换。
+ DDPMs 用于连续 domain。

两种模型都能通过更多的计算和数据提高性能。

这说明，结合这两种方法在生成连续数据（如语谱图或图像）时可能效果更好。

推理阶段，自回归模型将文本 token 序列转换为 speech token 序列，然后 DDPM 将这些 token 解码为语音。

### 将自回归+DDPMs 用于 TTS

结构如下：
![](image/Pasted%20image%2020241010172903.png)

训练以下神经网络：
1. 自回归 decoder，根据文本预测 speech token 的概率分布
2. 类似 CLIP 的对比模型，用于对自回归 decoder 的输出进行排序
3. DDPM，将 speech token 转换为 speech spectrogram

### 条件输入

TorToise 使用 speech conditioning 输入到自回归模型和 DDPM。speech conditioning 是 target speaker 的音频片段，通过 self-attention encoder 转换为 MEL spectrogram。自回归 generator 和 DDPM 的 conditioning encoder 与网络一起训练。

### TorToise Trick

大部分时间的训练过程中，DDPM 将离散 codes 转换为 MEL spectrograms。收敛后，对 DDPM 在自回归 latent space 上进行微调，而不是 speech codes。
> 原因在于，自回归 latent space 比离散 tokens 更丰富，通过在这个 latent space 上微调，提高了下游 diffusion model 的效率。

### CLVP

图像中的 CLIP 用于对生成模型的输出进行排序，选择最好的。同样的方法可以应用到语音：大多数 TTS 数据集都是音频片段和文本的配对。TorToise 中，训练 CLVP，用于对 AR model 的输出进行 re-ranking。

## 训练

在 8 个 NVIDIA RTX-3090s 上训练。

## 推理

推理过程如下：
1. 将 conditioning inputs 和文本输入到自回归模型，解码得到 output candidates
2. 使用 CLVP 计算 speech candidate 和文本的相关性分数
3. 选择 top k speech candidates，对每个 candidate：
4. 使用 DDPM 将其解码为 MEL spectrogram
5. 使用 vocoder 将其转换为 waveform
6. 解码自回归模型时，使用 nucleus sampling，P=0.8，repetition penalty=2，softmax temperature=0.8

TorToise TTS 的 DDPM 采样参数如下：
1. 算法：DDIM
2. Schedule：Linear
3. Sampling steps：64
4. Conditioning-Free Guidance constant：2

## 数据集

使用 LibriTTS 和 HiFiTTS 数据集，共 896 小时的 transcribed speech。另外从
互联网上爬取 49,000 小时的 speech audio 构建拓展数据集。验证集使用 LibriTTS 测试集。

## 实验

使用 CLVP 生成真实样本和生成样本之间的距离度量，类似于图像中使用的 FID 分数。使用开源 wav2vec 模型来表征 speech segment 的“可懂性”。
