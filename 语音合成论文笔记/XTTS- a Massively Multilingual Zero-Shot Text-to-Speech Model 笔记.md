> Interspeech 2024，Nvidia、Coqui.ai

1. 大部分 zero-shot TTS 系统只支持单一语言
2. 提出 XTTS，基于 Tortoise，实现多语言训练，提升语音克隆，加快训练和推理速度
3. 在 16 种语言上训练，且在大部分语言上达到 SOTA

> 本质为在 Tortoise 的基础上，增加多语种的数据实现多语言模型；通过单 codebook 和减少 code 实现推理加速；通过多个 embedding 提升语音克隆能力；通过 Speaker Consistency Loss 提升说话人相似度。

## Introduction

1. 现有的多语言 zero-shot TTS 系统支持的语种较少，且主要集中在中高资源语言
2. 提出 XTTS，支持 16 种语言：
    1. 为第一个支持低/中资源语言的多语言 zero-shot TTS 模型；
    2. 可以实现跨语言的语音克隆；
    3. 模型开源

## XTTS

XTTS 基于 [Tortoise TTS- Better speech synthesis through scaling 笔记](Tortoise%20TTS-%20Better%20speech%20synthesis%20through%20scaling%20笔记.md)，结构如图：
![](image/Pasted%20image%2020250808114149.png)

模型包含三个部分：
+ VQ-VAE：13M 参数，输入 mel 谱，只用 1 个 codebook，8192 个 codes，21.53 Hz 帧率
    + 结构和训练过程与 Tortoise 相同，但训练后过滤掉了不常用的 codes，只保留前 1024 个最常用的 codes
+ Encoder：GPT-2 encoder，443M 参数，输入文本 tokens，输出 VQ-VAE 的音频 codes
    + 额外通过一个 Conditioning Encoder 对 mel 谱进行编码，输出 32 个 1024 维的 embedding
    + 在多语言训练中，使用单个 embedding 会导致语音克隆能力下降，因此使用了多个 embedding
+ Decoder：基于 HiFi-GAN，26M 参数，输入来自 GPT-2 encoder 的 latent vectors
    + 由于 VQ-VAE 的高压缩率，直接从 VQ-VAE codes 重建音频会导致发音问题和伪影，因此使用 GPT-2 encoder 的 latent space 作为输入
    + 把 speaker embedding 作为解码器的条件
    + 引入 Speaker Consistency Loss 提高说话人相似度

VQ-VAE 和 encoder 使用 22.5 kHz 的音频信号进行训练，但 decoder 通过上采样到实际长度来生成 24 kHz 的音频。

## 实验（略）
