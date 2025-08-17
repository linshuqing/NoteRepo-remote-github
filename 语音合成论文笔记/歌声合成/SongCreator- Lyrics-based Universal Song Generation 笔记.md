> NIPS 2024，清华深研院、港中文

1. 提出 SongCreator，基于歌词生成歌曲，包含两部分：
    1. 双序列语言模型 (DSLM)，捕捉人声和伴奏
    2. 一系列注意力掩码策略，实现理解、生成和编辑歌曲
2. 在八个任务上实现了 SOTA 或者有竞争力的表现；可以独立控制人声和伴奏的声学条件

## Introduction

1. 歌曲包含人声和伴奏，AIGC 在歌曲生成上仍然是一个大问题
2. 目前没有结合人声和伴奏进行高质量的歌词到歌曲生成的模型，Jukebox 是唯一一个用单一模型从歌词生成包含人声和伴奏的歌曲的方法，但有两个主要限制：
    1. 将人声和伴奏的结合视为一个整体，忽略了两者之间的相互影响
    2. 仅限特定的歌词到歌曲生成任务
3. 提出 SongCreator，基于歌词生成包含同步协调的人声和伴奏的歌曲，包含：
    1. 双序列语言模型 (DSLM)，用两个解码器分别建模人声和伴奏，采用动态双向交叉注意力模块捕捉两个序列之间的影响
    2. 设计一系列注意力掩码策略，以统一方式实现理解、生成和编辑歌曲

## 相关工作（略）

## 方法

### 概览

定义 $x \in \mathcal{X}$ 为歌曲，$f : \mathcal{C} \to \mathcal{X}$ 为歌曲生成过程，其中 $\mathcal{C}$ 是条件信号的集合。本文 $C \in \mathcal{C}$ 包括歌词、人声提示、伴奏提示、预定人声轨和预定伴奏轨。

本文引入 semantic tokens $S = (S_1, \ldots, S_N)$ 捕捉歌曲中的结构信息，通过语言模型合成歌曲。

如图：
![](image/Pasted%20image%2020250811113329.png)

在包含歌曲、人声和音乐的 unlabeled 数据集上训练 BEST-RQ，然后对中间表征进行 VQ 得到 semantic tokens $S$。采用 LDM（由 VAE 和扩散模型组成）将 semantic tokens 解码为歌曲。

设计 DSLM 从条件 $C$ 预测 semantic tokens $S$，包含三个解码器，分别预测人声、伴奏和歌曲的 semantic tokens（即 $S_v$、$S_a$ 和 $S_s$），结构如图：
![](image/Pasted%20image%2020250811113903.png)
> 训练数据来源：对歌曲应用分离算法，得到配对数据。

### DSLM

DSLM 从条件 $C$ 生成 $(S_s, S_v, S_a)$，如图所示。DSLM 使用不同的解码器分别建模人声和伴奏的 semantic tokens，两者结合起来生成歌曲的 semantic tokens。

DSLM 包含 lyrics encoder、vocals decoder、accompaniment decoder 和最终的 song decoder。lyrics encoder 由 Transformer encoder 层堆叠而成，提取与歌词发音相关的信息。vocals decoder 和 accompaniment decoder 由多个 DSLM 块组成，每个块包含自注意力层（SA）、交叉注意力层（CA）、双向交叉注意力层（BCA）和前馈层。

CA 关注 lyrics encoder 的信息，对 vocal encoder，其建模歌词和人声之间的对齐；对 accompaniment decoder，其提取歌词的语义信息以生成伴奏。
BCA 用于理解和建模人声和伴奏之间的关系，包含两个对称的交叉注意力机制，例如，在 vocals decoder 中，BCA 使模型在生成人声时关注伴奏。定义如下：
$$\begin{gathered}{Q}_v={H}_v{W}_v^Q,\quad{K}_v={H}_a{W}_v^K,\quad{V}_v={H}_a{W}_v^V\\{M}_{ij}=\begin{cases}0,\quad\text{allow to attend}\\-\infty,\quad\text{prevent from attending}&\end{cases}\\{A}_v=\mathrm{softmax}\left(\frac{{Q}_v{K}_v^\top}{\sqrt{d_k}}+{M}\right)\end{gathered}$$
其中 $H_v, H_a \in \mathbb{R}^{T \times d_h}$ 分别表示来自 vocals decoder 和 accompaniment decoder 的前一层输出。通过权重 $W_v^Q, W_v^K, W_v^V \in \mathbb{R}^{d_h \times d_k}$ 投影为QKV，掩码矩阵 $M \in \mathbb{R}^{T \times T}$ 用于控制是否进行 attend，其中 $T$ 表示 LM 中 token 的长度，$d_h$ 和 $d_k$ 分别表示隐藏大小和注意力层大小。

vocals decoder 和 accompaniment decoder 逐 token 进行自回归预测。给定人声提示（用 semantic tokens 表示）$\hat{S}_v$，可以控制说话者、人声旋律和节奏；给定伴奏提示（用 semantic tokens 表示）$\hat{S}_a$，可以控制乐器、音乐旋律和节奏。提示音频的 semantic tokens 作为 prefix 输入 DSLM。以 vocals decoder $\theta_{vocal}$ 为例，其任务可表示为：
$$p({S}_v|{C}_{\mathrm{lyrics}},\hat{{S}}_v;\boldsymbol{\theta}_{\mathrm{vocal}})=\prod_{t=0}^Tp({S}_{v,t}|{S}_{v,<t},{S}_{a,<t},{C}_{\mathrm{lyrics}},\hat{{S}}_v;\boldsymbol{\theta}_{\mathrm{vocal}})$$

然后将两个解码器的输出 embedding $E_v, E_a \in \mathbb{R}^{T \times d_e}$ 拼接得到 $E_s \in \mathbb{R}^{T \times 2d_e}$，输入到由多个 Transformer 组成的 song decoder 中，非自回归地生成完整歌曲的 semantic token 序列，实现人声和伴奏的结合：
$$p({S}_s|{E}_v,{E}_a;\boldsymbol{\theta}_{\mathrm{song}})=\prod_{t\boldsymbol{=}0}^Tp({S}_{s,t}|{S}_v,{S}_a;\boldsymbol{\theta}_{\mathrm{song}})$$

### 注意力掩码策略

SA 和 SCA 都使用了掩码矩阵 $M$，不同的 $M$ 实现了不同的注意力机制。
+ 对于 SA，采用两种掩码策略：
    1. 因果注意力掩码：每个 token 只看其左边和自身，学习生成和延续能力，但难以捕捉上下文之间的依赖关系。
    2. 非因果注意力掩码：所有 token 都可以互相看到，从整个序列中获取上下文信息，生成全面丰富的上下文表征
+ 对于 BCA，设计了四种掩码策略：
    1. 双向掩码 (BR)：人声和伴奏序列的表征可以互相看到，但在预测时间步 $t$ 的 token 时，只能看到其他序列中小于等于 $t$ 的 token。
    2. 伴奏到人声 (A2V)：人声序列可以看到伴奏序列的全部上下文，而伴奏序列不能看到人声序列。
    3. 人声到伴奏 (V2A)：伴奏序列可以看到人声序列的全部上下文，而人声序列不能看到伴奏序列。
    4. 无 (None)：两个序列都不能互相看到，独立生成器乐或人声。

### 训练设置

采用多任务训练，包含：
+ 歌词生成歌曲：vocals decoder 和 accompaniment decoder 中的 SA 使用因果注意力掩码，同时生成人声和伴奏的 semantic tokens。BCA 80% 的时间使用双向注意力掩码，20% 的时间使用无掩码策略。
+ 给定伴奏或人声生成歌曲：以伴奏为例，vocals decoder 中的 SA 使用因果掩码生成人声，accompaniment decoder 中的 SA 使用非因果掩码，BCA 使用 A2V 策略。对于非因果掩码，随机 mask 输入序列中的 20% token，鼓励模型学习上下文 token 之间的关系。
+ 歌曲编辑：结合上述两种任务，随机选择目标序列末尾的一段 token 替换音频提示，在中间使用特殊 token `<EDIT>` 区分编辑任务和生成任务。

所有的训练任务中，vocals decoder 和 accompaniment decoder 使用 next token prediction 目标，song decoder 基于从 vocals decoder 和 accompaniment decoder 的 embedding 预测歌曲的 semantic tokens。通过人声、伴奏和歌曲的交叉熵损失优化 DSLM。
> 对所有 token 计算损失，而不仅仅是 mask 的 token。此外还随机 mask 歌词的 20%，鼓励模型尝试无条件生成

## 实验

数据：8,500 小时的带歌词的歌曲（约 270,000 首歌曲）；将数据集分割为 1.7M 个不超过 30 秒的片段，每个片段确保句子的完整性。每个片段通过 Demucs 音源分离提取人声和伴奏。

训练：8 * NVIDIA A800 GPU ，Adam 优化器，$\beta_1 = 0.9, \beta_2 = 0.98, \epsilon = 10^{-9}$。推理时采用 top-k 采样，$k$ 和 temperature 分别为 50 和 0.9。

评估：客观指标（FAD、MCD、SECS）和主观指标（MOS、AB 测试）

### 结果（略）
