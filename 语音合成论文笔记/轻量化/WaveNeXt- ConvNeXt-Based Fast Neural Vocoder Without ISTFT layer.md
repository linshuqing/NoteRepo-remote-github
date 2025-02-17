> ASRU 2023，NII、神户大学、名古屋大学

1. vocos 使用 ConvNeXt 预测 STFT 谱，使用 ISTFT 层合成波形，速度很快
2. 提出 WaveNeXt，用一个可训练的线性层替代 ISTFT 层直接预测波形来提高合成质量
3. 结合 JETS 的 E2E TTS 框架，可以构建 E2E TTS 模型
4. 实验结果表明 WaveNeXt 在保持速度的情况下比 vocos 合成质量更高

## Introduction
1. [HiFi-GAN- Generative Adversarial Networks for Efficient and High Fidelity Speech Synthesis 笔记](../HiFi-GAN-%20Generative%20Adversarial%20Networks%20for%20Efficient%20and%20High%20Fidelity%20Speech%20Synthesis%20笔记.md) 可以实现高质量合成，但是 RTF 为 0.7，不适合实时应用
2. 后续工作使用轻量级的快速上采样层替代 HiFi-GAN 的最后 4× 上采样层，提高速度；也有使用可训练的线性层替代 ISTFT 的上采样层，提高了速度和合成质量
3. vocos 使用 ConvNeXt 预测 STFT 谱，速度快，但是合成质量低
4. 提出 WaveNeXt，用一个可训练的线性层替代 ISTFT 层直接预测波形来提高合成质量；结合 JETS 的 E2E TTS 框架，构建 E2E TTS 模型；实验结果表明 WaveNeXt 在保持速度的情况下比 vocos 合成质量更高

## 基于 HiFi-GAN 的快速 neural vocoders（略）

## VOCOS

VOCOS 中，不需要上采样模块直接使用 ConvNeXt 预测 STFT 谱的幅度和相位，结构如上图 d。考虑输出的相位在 $(-\pi,\pi]$ 范围内，隐藏特征预测两个值，$m$ 和 $p$，其分别用来预测幅度和相位,$M=\operatorname{exp}(m),\phi=\operatorname{atan2}(\operatorname{sin}(p),\operatorname{cos}(p))$，然后复数 STFT 谱为 $M\cdot e^{j\phi}$。训练时使用了 HiFi-GAN 中的 MPD 和 UnivNet 中的 MRD，损失函数定义为：
$$\begin{aligned}&\mathcal{L}_{\mathrm{G}}=\ell_{\mathrm{G,MPD}}+w_{\mathrm{MRD}}\ell_{\mathrm{G,MRD}}\\&+\ell_{\mathrm{FM,MPD}}+w_{\mathrm{MRD}}\ell_{\mathrm{FM,MRD}}+w_{\mathrm{mel}}\ell_{\mathrm{G,mel}}\\&\mathcal{L}_{\mathrm{D}}=\ell_{\mathrm{D,MPD}}+w_{\mathrm{MRD}}\ell_{\mathrm{D,MRD}},\end{aligned}$$
其中 $\ell_{G,MPD},\ell_{G,MRD},\ell_{D,MPD},\ell_{D,MRD}$ 分别为 MPD 和 MRD 的生成器和判别器的对抗损失函数，$\ell_{FM,MPD},\ell_{FM,MRD}$ 为 MPD 和 MRD 的特征匹配损失函数，$\ell_{G,mel}$ 为 mel-spectrogram L1 损失，$w_{MRD},w_{mel}$ 为权重系数。Vocos 使用了 hinge loss。

## WaveNeXt

不同模型的结构对比：
![](image/Pasted%20image%2020250215150753.png)


### iSTFT 层的上采样是否真的有必要？


![](image/Pasted%20image%2020250215151824.png)

上图为 Vocos 预测的 STFT 谱的幅度和相位与 GT 的对比，可以看到预测的幅度和相位与真实的有一定差距，但是合成的波形质量还是很高的，原因在于重叠相加操作和 GAN 训练的冗余性。因此，直接在时域预测波形比在频域预测 STFT 更适合 GAN 训练。

### WaveNeXt

WaveNeXt 使用可训练的线性层替代 Vocos 中的 ISTFT 层，结构如上图 e。输入和输出通道为 FFT 长度和 shift 长度，最终的线性层直接预测波形，维度为 $l_{shift}\times T$，最后合成波形长度为 $1\times l_{shift}T$。这里相当于是 256x 的上采样，但是由于 ConvNeXt 层和可学习的线性层，WaveNeXt 可以在保持速度的情况下比 Vocos 合成质量更高。

### JETS-based E2E TTS 模型

提出使用 Vocos 和 WaveNeXt 的 E2E TTS 模型。

原始 [JETS- Jointly Training FastSpeech2 and HiFi-GAN for End to End Text to Speech 笔记](JETS-%20Jointly%20Training%20FastSpeech2%20and%20HiFi-GAN%20for%20End%20to%20End%20Text%20to%20Speech%20笔记.md) 联合训练 FastSpeech 2 和 HiFi-GAN，不需要 mel 谱特征和外部对齐工具。JETS 的 generator 结构如下图：
![](image/Pasted%20image%2020250215153617.png)

最终基于 Vocos 和 WaveNeXt 的 E2E TTS 模型的损失函数为：
$$\begin{aligned}\mathcal{L}_{\mathrm{G,E2ETTS}}&=\mathcal{L}_{\mathrm{G}}+w_{\mathrm{var}}\ell_{\mathrm{var}}+w_{\mathrm{align}}\ell_{\mathrm{align}}\\\mathcal{L}_{\mathrm{D,E2ETTS}}&=\mathcal{L}_{\mathrm{D}},\end{aligned}$$
其中 $\ell_{var}$ 和 $\ell_{align}$ 分别为 JETS 中的 variance loss 和 alignment loss，$w_{var}$ 和 $w_{align}$ 为权重系数。

### Full-band E2E TTS 模型

为了进一步提高合成质量，提出 48 kHz 的 full-band E2E TTS 模型。ConvNeXt 模型可以通过改变 FFT 和 shift 长度来实现 full-band 合成。与 HiFi-GAN 不同，ConvNeXt 模型的推理速度几乎与 48 kHz 的相同。

## 实验（略）
