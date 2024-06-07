
## 特点

zero shot TTS：5s 参考音频

few shot TTS：1min 训练数据

webui 支持

语言：中日英，可以跨语言转换

## 微调

1. 下载预训练模型
2. 下载 ASR 模型
3. 数据预处理：
	1. 用的数据为 干声（即无伴奏人声，有伴奏的话用 UVR5 去除）
	2. 填路径、分割语音，得到 slice 文件
	3. 进行 ASR 识别，得到 list 文件
	4. 校对 list
	5. 填入 list 和 音频 文件路径，预处理，得到文件：
		1. 2-xxx：name2text.txt
		2. 3-xxx：SSL 特征，pt 后缀
		3. 4-xxx：中文 SSL 特征，pt 后缀
		4. 5-xxx：音频
        5. 6-xxx：name2semantic.tsv，SSL 特征通过量化器得到的离散的 code

## 从 SoVITS 到 GPT-SoVITS
SoVITS 做的是 VC，不做 TTS，因此输入只有两端音频而没有文本。

采用 SoftVC 作为 先验编码器，目的是去除音色信息（speaker）保留内容或语义信息。
得到的语义特征再通过 VITS 进行语音合成
> 其实就是用这个特征作为 VITS 中的 condition，即替换了原始的文本特征

推理的时候，需要提供目标说话人的音色信息。
> 训练的时候当然也会，只不过训练的时候因为没有并行数据，所以用的是同一个人的语音（同一个人的音色）。推理的时候，需要提供目标说话人的音色信息，这样就可以实现说话人转换。

但是，采用 SoftVC 作为 先验编码器 来去除音色的时候，并不能完全去除。
> 这也是后续更新为什么用 ContentVec 这个模型的原因。

对 GPT-SoVITS 的启发：
1. 如果泄漏的音色较多（或者说 SSL 特征包含的音色特征较多），那是不是干脆用一个泄漏音色很大的模型（如 CN_HuBERT）作为一种 “引导”，从而缺点变成优点，这个 SSL 特征保留了丰富的音色（所谓的 音色丰富的 语义 token）
然后用 GPT 模型来 “补全” 这些特征。

考虑 VC 任务，此时只有目标语音作为输入：
+ 从目标语音的 SSL 特征中提取带有音色信息的 SSL 特征
+ 用 ContentVec 模型从源语音中提取语义特征
+ 然后把这两个特征通过 GPT 模型进行预测，得到预测的 token，此时的 token 同时包含了目标语音的音色信息和源语音的语义信息

然后考虑 TTS 任务，此时会给定文本输入：
+ 此时我们有两个特征，
    + 一个是前面说的带有音色信息的 SSL 特征
    + 另一个是 从要合成的文本得到的文本语义特征（注意：TTS 任务中，前面说的 ContentVec 模型就不需要了）。
    + （可选）实际使用时，还会加上参考音频的文本（参考音频就是用于提取 SSL 的那个音频）
+ GPT 模型的输入就是前面的两个或者三个特征，预测得到的输出是，目标音色下，对应于输入文本的 token，也就是同时包含一定的目标音色信息和要合成的文本的语义信息的 token。
+ 这样子模型就相当于有两部分的音色，一个是 GPT 生成的 SSL 中 “继承” 的音色，另一个是在 z_p 特征中手动给模型的音色特征。
> 这样可以缓解 VITS 重构音色的压力，之前的音色都是手动给的，现在相当于通过 SSL 又多给了。

具体来说：
整体模型框架：ar-vits
GPT 模型用的是 SoundStorm
音色编码器用的是 TransferTTS
中文的 SSL 特征用的是腾讯游戏的 CN_HuBERT
VC 任务中用上了 ContentVec

> GPT-SoVITS 约等于 VALLE+VITS

## 模型结构
微调包括两个部分，且两个部分是无序的
1. SoVITS，对应代码 train_s2
2. GPT 模块，对应代码 train_s1

微调的时候，先数据预处理得到上述需要的特征：
然后：
ssl 特征通过量化器得到 code，这些 code 后续会用于训练 GPT 模块。
> 这里的 code 包含了 内容信息 和 speaekr 信息

code 和 文本特征 输入到 先验编码器TextEncoder 得到分布的均值和对数方差。

然后通过 flow 得到 z_p，剩下的和 VITS 一样。
对于 后验编码器PosteriorEncoder，输入的是音频的 mel 谱 特征

## 代码

### 数据预处理

### 微调

train_s1：训练 GPT
输入：text1 + text2 + wav1-hubert-token
输出：wav2-hubert-token
> 本质就是一个 VALLE

train_s2：训练 VITS
输入：wav2-hubert-token + wav1-audio（用于提取音色信息的） + text2
输出：wav2-audio
> 这里还多加了一个 s1 得到的 wav2-hubert-token，也就是前面反复提到的泄漏了音色的 SSL 特征。

> 注意：wav1 reference wav，即用来给音色的，wav2 为 target wav，即要合成的

train_s2：
SynthesizerTrn：来自 VITS 的 generator
freeze_quantizer = True

输入：
+ ssl
+ y：mel 谱
+ y_lenght
+ text：文本
+ text_length


train_s1：
用 Pytorch Lightning 框架
模型：Text2SemanticLightningModule
+ 输入：phoneme_ids、phoneme_ids_len、semantic_ids、semantic_ids_len、bert_feature
+ 内部的模型为 Text2SemanticDecoder
    + 输入为 phoneme_ids、phoneme_ids_len、semantic_ids、semantic_ids_len、bert_feature
    + forward 中，make_input_data 用于准备输入和输出数据：
        + xy_pos：就是对齐之后的用于 GPT 的输入
        + target：GPT 模型的目标
    + infer 的时候，semantic_ids 就变成了 prompt，
+ forward 函数在 AR/models/t2s_model.py 中




数据：Text2SemanticDataModule
+ 包含 train 和 val
+ 用的是 DistributedBucketSampler
+ dataset 用的是 Text2SemanticDataset，有 collate_fn
+ __getitem__ 返回的是
    "idx": idx,
    "phoneme_ids": phoneme_ids,
    "phoneme_ids_len": phoneme_ids_len,
    "semantic_ids": semantic_ids,
    "semantic_ids_len": semantic_ids_len,
    "bert_feature": bert_feature,
+ 

## 推理

推理对应的代码为：
inference_webui 为推理的窗口


## 问题

GPT 模型具体的输入输出是什么？