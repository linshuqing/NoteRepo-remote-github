> Preprint 2025，bilibili

1. 自回归的 TTS 模型逐个生成 token，但是很难精确控制合成语音的时长
2. 提出 IndexTTS2，包含两种生成模式：
    1. 显式指定生成 token 数量，精确控制语音时长
    2. 不需要手动输入 token 数量，模型自回归生成语音
3. 模型可以实现情感表达和说话人身份的解耦，独立控制音色和情感
4. 引入 GPT latent 表征来提高强情感表达下语音的稳定性；通过微调 Qwen3 设计 soft instruction 机制，使用自然语言来指导语音生成的情感倾向

## Introduction

1. 生成式 TTS 模型主要分为自回归和非自回归两种模式：
    + 自回归模型在自然性和表现力上很好，但由于其逐个生成 token 的特性，难以精确控制合成语音的时长
    + 非自回归模型可以通过并行解码实现快速推理，并语音时长等参数可控
2. 情感可控的 TTS 模型主要有两类：
    + 用训练数据中的情感标签或通过 CLAP 将自然语言转换为情感 embedding
    + 通过参考音频增强情感表达

3. 本文提出 IndexTTS2，如下图，模型可以实现固定时长语音生成，且以自回归方式生成；模型可以通过自然语言控制情感
![](image/Pasted%20image%2020250808210532.png)

4. IndexTTS2 包含三个模块：
    + T2S：采用基于 transformer 的自回归模型，从文本、timbre prompt 和 style prompt 生成 semantic token 和语音 token 数量（可选）；从 style prompt 中提取情感特征，并使用梯度反转层消除与情感无关的信息
    + S2M：输入 semantic token 和 timbre prompt，生成 mel 频谱图；引入 GPT latent 表征来提高生成语音的稳定性
    + Vocoder：使用 BigVGANv2 将 mel 频谱图转换为波形

5. 本文贡献如下：
    + 提出了一种新颖的自回归 TTS 模型 IndexTTS2，第一个实现精确时长控制的自回归 zero-shot TTS 模型
    + 解耦情感和说话人相关特征，设计特征融合策略以保持语义流畅性和发音清晰度；实现基于自然语言描述的情感控制

## 相关工作（略）

## IndexTTS2

### 自回归 T2S

结构如图：
![](image/Pasted%20image%2020250809104918.png)

首先构造序列 $[c,p,e_{\langle BT\rangle},E_{text},e_{\langle BA\rangle},E_{sem}]$，其中 $c$ 是表示说话人相关或情感的全局条件，$p$ 是用于时长控制的 embedding，$E_{text}$ 是文本 embedding 序列，$E_{sem}$ 是 semantic token embedding 序列，$e_{\langle BT\rangle}$ 和 $e_{\langle BA\rangle}$ 分别是文本和语义 token 序列前的特殊 token embedding。训练目标是预测目标 semantic token 序列。T2S 模块的结构与 [IndexTTS- An Industrial-Level Controllable and Efficient Zero-Shot Text-To-Speech System 笔记](IndexTTS-%20An%20Industrial-Level%20Controllable%20and%20Efficient%20Zero-Shot%20Text-To-Speech%20System%20笔记.md) 类似。
> 多了两个 embedding：分别用于时长控制和情感控制

#### 自回归系统的精确语音时长控制

训练时，对于每对 (text, speech)，说话人（timbre）prompt 从同一说话人的另一个语音中随机选择。输入文本被编码为 BPE token 序列，然后通过文本 embedding 表映射为 $E_{text} \in \mathbb{R}^{L \times D}$，其中 $L$ 是文本长度，$D$ 是 embedding 维度。

semantic token embedding 序列为 $E_{sem} \in \mathbb{R}^{T \times D}$，其中 $T$ 是语 semantic token 序列的长度。可学习的 position embedding table $W_{sem}$ 生成 positional embedding 序列 $P_{sem}^l = [p^1_{sem}, p^2_{sem}, ..., p^T_{sem}]$：
$$p_{sem}^l=W_\mathrm{sem}h(l),\quad l\in\{1,2,\ldots,T\}$$
其中 $W_{sem} \in \mathbb{R}^{L_{speech} \times D}$，$L_{speech}$ 是 semantic token 序列的最大长度。函数 $h(l)$ 表示位置 $l$ 的 one-hot 向量，在第 $l$ 个元素处为 1，其余为 0。

为了控制时长，在训练序列中引入一个特殊的 embedding $p$，其通过一个 embedding table $W_{num}$ 将 semantic token 序列 $\{y_t\}_{t=1}^T$ 的长度 $T$ 映射为向量 $p$：
$$p=W_{num}h(T)$$
> 在自回归模式下，为了实现时长控制，需要共享 $W_{sem}$ 和 $W_{num}$，即 $W_{sem} = W_{num}$。

实际发现使用两个不同的系数 $r_1$ 和 $r_2$ 对 gt 语音和说话人 prompt 进行随机速度调整，可以提高时长控制的准确性。同时，为了支持时长控制和自由生成两种模式，在训练过程中，$p$ 有 30\% 的概率被设置为零向量。

训练时最小化 semantic code 的交叉熵：
$$L_{AR}=-\frac{1}{T+1}\sum_{t=0}^T\log q(y_t)$$
其中 $y_T$ 表示 "end of semantic code sequence" token $\langle EA \rangle$，$q(y_i)$ 表示后验概率。

#### 情感控制

通过两阶段训练策略实现情感控制：
+ 阶段一：仅使用情感数据训练，使用基于 Conformer 的 emo perceiver conditioner 从 style prompt 中提取情感 embedding $e$，引入梯度反转层以解耦情感和说话人相关属性（如 accent、rhythm）
> 训练时，style prompt 来源于 GT 语音；推理时可以替换为情感参考音频（可能来自不同说话人）

训练时的序列为 $[c + e, p, e_{\langle BT \rangle}, E_{text}, e_{\langle BA \rangle}, E_{sem}]$。

引入辅助说话人识别损失：
$$L_{AR}=-\frac{1}{T+1}\sum_{t=0}^T\log q(y_t)-\alpha\log q(e)$$
其中 $q(e)$ 表示情感 embedding $e$ 对应于 GT 说话人的后验概率，$\alpha$ 是损失系数。

+ 阶段二：在大规模中性语音数据上微调模型，同时保持情感 perceiver conditioner 不变。

> 定义七种标准情感并构建了相应的情感 embedding 数据库，通过自然语言给定所需的情感，经过一个微调好的 LLM 处理后估计情感概率分布。最后通过加权标准向量得到最终的情感向量作为模型的输入。

### S2M

如图：
![](image/Pasted%20image%2020250809102704.png)

S2M 为基于 flow matching 的非自回归生成框架，使用 CFM 学习 ODE 模型将噪声分布变为目标 mel 谱分布。条件为 timbre reference audio 和 T2S 生成的 semantic code。为了解决语音模糊问题，采用了两种策略：
+ 策略一：使用 BERT 提取的文本表征作为辅助输入
+ 策略二：使用 T2S 的 GPT latent 特征作为辅助信息

给定 timbre reference speech $u_{tim}$ 和其 mel 谱 $y_{tim}$，目标是重建同一说话人的目标 mel 谱 $y_{tar}$。

训练时输入为纯噪声。T2S 生成的 semantic token 为 $Q_{sem}$，T2S 的 latent 表征为 $H_{gpt}$。为了增强模型的鲁棒性，使用 MLP 以 50\% 概率随机将 $H_{gpt}$ 和 $Q_{sem}$ 相加，得到最终增强的 semantic token $Q_{fin}$。speaker embedding 通过 perceiver conditioner 提取，并与增强后的 semantic token 拼接作为条件。最后得到的 $Q_{fin}$ 作为 S2M 模块的输入。模型通过最小化 $y_{pred}$ 和原始 Mel 谱 $y_{tar}$ 之间的 L1 损失进行训练：
$$\begin{aligned}\mathcal{L}_{L1}&=\frac{1}{F\cdot D}\sum_{f=1}^{F}\sum_{d=1}^{D}|(y_{pred})_{f,d}-(y_{tar})_{f,d}|\end{aligned}$$

其中 $F$ 是 Mel 谱的总帧数，$D$ 是 Mel 频率维度。

## 实验和结果（略）
