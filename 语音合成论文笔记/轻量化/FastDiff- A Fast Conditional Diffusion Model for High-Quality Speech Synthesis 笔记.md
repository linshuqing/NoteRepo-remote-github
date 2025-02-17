> IJCAI 2022，浙江大学，腾讯 AI Lab，Dong Yu，Zhou Zhao

1. DDPM 的迭代采用非常耗时
2. 提出 FastDiff，通过堆叠不同 receptive field patterns 的 time-aware location-variable 卷积层来自适应地建模长期依赖，同时采用 noise schedule predictor 来减少采样步数而不牺牲质量
3. 基于 FastDiff 提出 FastDiff-TTS，可以得到 SOTA 的生成效果，在 V100 GPU 上比 real time 快 58 倍

## Introduction

4. 语音生成的两个目标：高质量+快速
5. DDPM + 语音合成当前有两个挑战：
	1. DDPM 其本质由于是迭代降噪，而不管生成效果和最参考语音的差异，可能降噪过度导致语音中的一些呼吸声被去掉，反而影响真实度
	2. 降噪过程需要好多迭代步，如果减少步数效果又不好
6. 本文提出 FastDiff，通过堆叠不同 receptive field patterns 的 time-aware location-variable 卷积层来自适应地建模长期依赖，同时采用 noise schedule predictor 来减少采样步数进行推理加速而不牺牲质量
7. 最终 MOS 值更高，只要四次迭代就可以实现高质量合成

## 背景 - DDPM（略）

## FastDiff

![](image/Pasted%20image%2020230527105114.png)
### Motivation

在工业界，DDPM 的部署还存在一些挑战：
+ diffusion catch dynamic dependencies from noisy audio instead of clean ones，which introduce more variation information
+ 当 receptive field patterns 有限时，反向迭代减少时可能出现性能退化；而迭代次数过多又难以部署

提出两个方法：
+ 采用 time-aware location-variable ，捕获噪声样本细节
+ 采用 noise schedule predictor 减少反向迭代次数

### Time-Aware Location-Variable Convolution

LVCNet 提出 location-variable convolution，可以有效地建模音频的长期依赖。

受此启发提出 Time-Aware Location-Variable Convolution，和 diffusion 中的 time step 也有关系。在 time step $t$，按照 Transformer 中 positional embedding 计算方法，对 time 进行 embed 到 128 维的 positional encoding vector $e_t$：$$\begin{aligned}
\boldsymbol{e}_t= & {\left[\sin \left(10^{\frac{0 \times 4}{63}} t\right), \cdots, \sin \left(10^{\frac{63 \times 4}{63}} t\right),\right.} \\
& \left.\cos \left(10^{\frac{0 \times 4}{63}} t\right), \cdots, \cos \left(10^{\frac{63 \times 4}{63}} t\right)\right],
\end{aligned}$$
在 TALVC 中，FastDiff 需要多个预测的 variation-sensitive kernels 来对输入序列的相关区间进行卷积，这些 kernel 应该是 time-aware 并且对噪声音频的变化（包括不同的 time step 和 声学特征）敏感。因此提出 time-aware 的 LVC 模块，如图 1-c。

对于第 $q$ 个 time-aware LVC 层，采用长为 $M$ dilation 为 $3^q$ 的窗口，把输入 $\boldsymbol{x}_t\in\mathbb{R}^D$ 分成 $K$ 个段，每段 $\boldsymbol{x}_t^k\in\mathbb{R}^M$：$$\left\{\boldsymbol{x}_t^1, \ldots, \boldsymbol{x}_t^K\right\}=\operatorname{split}\left(\boldsymbol{x}_t ; M, q\right)$$
然后，kernel predictor $\alpha$ 会生成 kernels，用这些 kernels 对输入序列的相关区间进行卷积运算：$$\begin{aligned}
\left\{\boldsymbol{F}_t, \boldsymbol{G}_t\right\} & =\alpha(t, c) \\
\boldsymbol{z}_t^k & =\tanh \left(\boldsymbol{F}_t * \boldsymbol{x}_t^k\right) \odot \sigma\left(\boldsymbol{G}_t * \boldsymbol{x}_t^k\right) \\
\boldsymbol{z}_t & =\operatorname{concat}\left(\left\{\boldsymbol{z}_t^1, \ldots, \boldsymbol{z}_t^K\right\}\right),
\end{aligned}$$
由于 time-aware 的 kernel 可以适应不同的 noise-level，并且和 声学特征 相关，从而可以精确估计降噪过程的梯度，且速度很快。
> 其实也非常简简单，在 LVCNet 中的 kernel predictor 只和 $c$ 有关，但是在 diffusion 中存在多个迭代的 time step $t$，所以很自然就将  kernel predictor 同时也看成是 $t$ 的函数，也就是在不同的迭代步（不同 time step 本质就是不同的 noise  level）下有不同的 kernel。

### 加速采样

#### Noise Predictor

采用了 BDDMs 中的 noise scheduling 算法来提高速度。

#### Schedule Alignment

训练 FastDiff 时候，采用 $T=1000$ 个 time step，在采样的时候需要基于 $t$，还要通过将 $T_m$ step 下的 sampling noise schedule 对齐到 $T$ step 下的 training noise schedule。

### 训练、Noise Scheduling and Sampling

