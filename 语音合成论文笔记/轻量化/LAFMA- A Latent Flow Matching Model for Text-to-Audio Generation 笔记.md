> Interspeech 2024，厦大、快手、南开

1. 之前的 TTA 方法用 latent space 下的 diffusion 模型，diffusion 模型的质量、生成效率有待提高
2. 本文探索在 latent space 中集成 FM 模型，提高生成音频质量，减少推理步数

> 其实就是在 latent space 中使用 FM 模型，其他的和 diffusion 完全一样。

## Introduction

1. 现有的 TTA 可以分为两类：
    + 将音频转换为 token 序列，使用 transformer 自回归生成
    + 使用 diffusion 生成音频 mel-spectrograms
2. 基于 LDM 的音频生成模型效果很好，但是需要大量采样步
3. 提出 LAFMA，将 FM 集成到音频 latent space 中进行音频生成，基于 [SpeechFlow- Generative pre-training for speech with flow matching 笔记](../../语音自监督模型论文阅读笔记/SpeechFlow-%20Generative%20pre-training%20for%20speech%20with%20flow%20matching%20笔记.md) 中的预训练模型，贡献如下：
    + 提出 LAFMA，用 ODE solver 生成高质量音频样本
    + 探索在 latent FM 模型中使用无分类器引导，提高效果
    + LAFMA 在显著减少推理步数的同时取得了显著性能

## Flow Matching


给定数据分布 $p_1(x_1)$ 和高斯分布 $p_0(x_0)$，FM 模型直接建模概率路径 $p_t(x_t)$，考虑 ODE 方程：
$$dx_t = v_t(x_t)dt$$
其中 $v_t$ 表示向量场，$t \in Uniform[0,1]$，ODE 与概率路径 $p_t(x_t)$ 通过连续性方程关联，通过神经网络估计得到 $v_t$ 后可以生成真实数据样本。

FM 训练时，从数据分布中抽取样本 $x_1$，采样 random flow step，计算 $x_t$ 和其导数 $v_t$ 的 noisy 版本，FM 模型 $u_\theta$ 基于 $t$ 和 $x_t$ 预测导数 $v_t$。推理时，从先验分布中抽取样本 $x_0$，用 ODE solver 估计 $x_1$，通过对导数积分得到 $x_1$。

另一种方法是 optimal transport (OT)，使用 constant directions 和 speeds 的条件路径，OT 路径更容易学习。

对于样本 $x_1$ 和 flow step $t$，OT 的条件路径为 $x_t = (1-(1-\sigma_{min})t)x_0 + tx_1$ 和 $v_t = x_1-(1-\sigma_{min})x_0$，$x_0$ 从先验分布中抽取，$\sigma_{min}$ 是一个小值。FM 的目标定义如下：
$$\hat{\theta} = \arg \min_\theta E_{t,x_t} ||u_\theta(x_t,t)-v_t||_2$$

## LAFMA

LAFMA 如图：
![](image/Pasted%20image%2020250210145613.png)

包含：
+ text encoder：编码输入音频描述
+ latent flow matching model (LFM)：生成音频的 latent representation
+ mel-spectrogram VAE：基于 latent 音频表征重构 mel-spectrogram，得到的 mel-spectrogram 通过预训练的 vocoder 生成最终音频

### Text Encoder

使用预训练的 LLM FLAN-T5-Large 作为 text encoder，FLAN-T5 在大规模 COT 和 instruction-based 数据集上预训练，能够有效利用上下文信息，模拟 attention 权重的梯度下降，有助于学习新任务。同时可以增强得到的文本信息的质量，有助于生成准确的音频输出。

### 用于 TTA 的 Latent Flow Matching 模型

给定输入样本 $x_1 \sim p_1$，使用预训练的 VAE encoder 将其编码为 latent representation $z_1$。在 latent space 中，目标是估计从随机噪声 $z_0 \sim p_0$ 到 latent representation $z_1$ 的源分布的概率路径。

使用与 vanilla flow matching 相同的目标，假设恒定速度，同时加入额外的条件文本信息 $c$。优化目标如下：
$$\hat{\theta} = \arg \min_\theta E_{t,z_t} ||u_\theta(z_t,t,c)-v_t||_2$$

使用 Euler ODE Solver 进行迭代采样，得到预测的 latent representation $z$，然后输入 VAE decoder 生成 mel-spectrogram，最后通过预训练的 vocoder 将 mel-spectrogram 转换为音频波形。

### 无分类器引导

在条件向量场 $u_\theta(z_t,t,c)$ 中引入无分类器引导，训练过程中，以固定概率（10%）随机丢弃条件，同时训练条件 LFM $u_\theta(z_t,t,c)$ 和无条件 LFM $u_\theta(z_t,t)$，确保框架在不同条件下生成高质量音频输出。

在生成过程中，使用无分类器引导采样：
$$u_\theta(z_t,t,c) = wu_\theta(z_t,t,c) + (1-w)u_\theta(z_t,t)$$

其中 $w$ 表示引导比例，无分类器引导可以生成多样且可控的音频样本，实现生成质量和样本多样性之间的权衡。

### VAE 和 Vocoder

音频 VAE 将音频 mel-spectrogram $m \in \mathbb{R}^{T \times F}$ 压缩为音频表示 $z \in \mathbb{R}^{C \times T/r \times F/r}$，其中 $T,F,C,r$ 分别表示时间维度、频率维度、通道数、压缩级别。编码器和解码器由堆叠的卷积模块组成，通过最大化证据下界和最小化对抗损失进行训练。在实验中，$C$ 和 $r$ 分别设置为 8 和 4。使用 HiFi-GAN 作为 vocoder，从 mel-spectrogram 合成音频波形。

## 实验（略）
