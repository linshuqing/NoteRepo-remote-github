> Preprint 2025，bilibili

1. 提出 IndexTTS，基于 XTTS 和 Tortoise 改进：
    + 中文场景下，将字和拼音混合建模，使多音字和长尾字的发音可控
    + 对比了 VQ 和 FSQ 在 codebook 利用率的差异
    + 引入基于 Conformer 的语音条件编码器，将解码器替换为 BigVGAN2
2. 相比 XTTS，在自然度、内容一致性和 zero-shot 语音克隆上有提升
3. 相比 Fish-Speech、CosyVoice2、FireRedTTS 和 F5-TTS，训练过程更简单可控，推理速度更快，性能更好

> "字和拼音混合建模" 其实就是训练的时候既有汉字又有拼音；

## Introduction
1. 生成式 TTS 可以分为三类：
   + codec LM：通常使用多 codebook 和高帧率，简单但训练和推理时间长且不稳定
   + 基于 diffusion 的 TTS：非自回归，生成质量高但难以实现流式
    + 混合架构：使用单 codebook 和低比特率 codec，通过独立解码器生成高质量音频，平衡性能和生成质量
2. IndexTTS 基于 XTTS 和 Tortoise，做了以下改进：
    + 去掉 G2P，使用原始文本和基于 BPE 的 tokenizer 作为输入，简化预处理，便于多语言扩展
    + 提出字-拼音混合建模，使多音字和低频字的发音可控
    + 对比了 VQ 和 FSQ 在声学 token 的 codebook 利用率

## IndexTTS

和 [XTTS- a Massively Multilingual Zero-Shot Text-to-Speech Model 笔记](XTTS-%20a%20Massively%20Multilingual%20Zero-Shot%20Text-to-Speech%20Model%20笔记.md) 类似，IndexTTS 包含 VQ-VAE codec、text-to-codec LM 和 latent-to-audio decoder，如图所示：
![](image/Pasted%20image%2020250808165551.png)

### Text tokenizer

系统仅支持中英文，使用原始文本作为输入，通过基于 BPE 的 tokenizer 进行分词。中文场景下，采用字-拼音混合建模。tokenizer 的词表大小为 12000，包括 8400 个汉字及其对应的 1721 个拼音、英文词片和一些特殊符号。在训练时，随机选择一定比例的非多音字替换为拼音。

### Speech tokenizer

分析了 VQ 和 FSQ 的 codebook 利用率。将 VAE 的参数增加到约 50M，输入 mel 谱，每帧使用约 8192 个 codes 进行 VQ 编码，输入音频采样率为 24 kHz，输出 token 为 25 Hz。

### LLM

text-to-codec LLM 基于 decoder-only transformer，类似 XTTS。从输入的文本 token 生成音频 mel token。此外，还引入 Conformer 条件编码器以提升音色相似度和训练稳定性。

 LLM 的训练过程可分为以下几类（[BT] [ET] 表示文本 token 序列的开始和结束，[BA] 和 [EA] 表示音频 token 序列的开始和结束）：
+ SEQ1：“[BT], prompt_text, text, [ET], [BA], prompt_audio, audio, [EA]”，如 Vall-E 和 Fish-Speech，将提示和目标的所有 token 连接起来
+ SEQ2：“[BT], text, [ET], [BA], audio, [EA]”，如 CosyVoice2，直接从文本 token 生成音频 token
+ SEQ3：“speaker info, [BT], text, [ET], [BA], audio, [EA]”，如 XTTS、CosyVoice 和 Tortoise，将提示音频的说话人信息压缩为一个或 32 个 latent vectors，作为 LLM 的条件

SEQ1 和 SEQ2 在推理过程中需要与 prompt audio 对应的文本，推理输入前缀序列可以构造为 “[BT], prompt_text, text, [ET], [BA], prompt_audio”。相比之下，SEQ3 只需要提示音频，推理输入前缀序列为 “speaker_info, [BT], text, [ET], [BA]”。从该输入前缀序列开始自回归生成 LM，直到检测到 “[EA]”。
> 相当于前两个是 zero-shot TTS，第三个是指定说话人的 TTS

本文采用 SEQ3。在跨语言语音克隆等场景下，不依赖提示文本非常重要。如果必须提供提示文本或通过多语言 ASR 系统进行识别，会限制其可用性。
> 因为前两个需要 prompt audio 对应的文本

相比于 Tortoise 和 CosyVoice 等单说话人编码向量，或 Vall-E 等语音提示方法，基于 Conformer 的 Perceiver 在捕捉说话人特征方面表现更优。

### Speech decoder

最后将 LLM 的输出转为波形。本文采用基于 BigVGAN2 的 vocoder，直接根据 LLM 的最后隐藏状态重建音频，并以 speaker embedding 作为条件。latent 采样率为 25Hz，上采样到 100Hz 后输入 BigVGAN2，最终输出 24KHz 的音频。

## 实验（略）
