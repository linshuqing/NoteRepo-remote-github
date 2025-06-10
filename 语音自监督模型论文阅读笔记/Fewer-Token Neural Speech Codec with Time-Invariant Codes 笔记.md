> ICASSP 2024，自动化所、清华

1. VALL-E 可以在 zero-shot 场景中实现 in-context learning，但是 codec 生成的 token 过多会影响预测准确性
2. 提出 TiCodec，将 time-invariant 信息编码为单独的 code，减少需要编码的 frame-level 信息，减少 token 数量
3. 引入 time-invariant encoding consistency loss，增强 utterance 中 time-invariant code 的一致性
4. 实验结果表明，TiCodec 可以减少 token 数量，提高语音质量，增加相似度和自然度，降低 TTS 模型合成语音的错误率

## Introduction

1. VALL-E 等模型需要多个 frame-level token 序列来重构高质量语音，影响推理速度和鲁棒性
2. 现有 codec 在使用一个或两个离散 token 序列表示语音时性能显著下降，无法重构高质量语音
3. 提出 TiCodec，将 time-invariant 信息提取并编码为固定长度的 code，减少需要编码的信息量，从而减少 frame-level token 数量；然后分开量化为 frame-level token 和 time-invariant token（TTS 中，time-invariant token 从目标说话人语音中提取）：
    + 引入 time-invariant encoding consistency loss，增强 time-invariant token 的全局一致性

## 方法

结构如图：
![](image/Pasted%20image%2020250605111651.png)


### 框架

整体框架是编解码器架构，使用 U-net-like 连接将编码器的中间层表示传递给解码器。
定义持续时间为 $d$ 的语音信号为 $x \in \mathbb{R}^T$，采样率为 $f_{sr}$，则 $T = f_{sr} \times d$。编码器 $Enc$ 将输入语音信号 $x$ 转换为表征 $z$，然后通过 RVQ $Q_1$ 进行量化。编码器的中间层表征 $h$ 输入 time-invariant representation extraction module（TIRE）以提取 time-invariant 表征 $m$，然后通过 GVQ $Q_2$ 进行量化。$Q_1$ 生成压缩表征 $z_q$ 和离散 token 序列 $c_f$，$Q_2$ 生成压缩表征 $m_q$ 和离散 token 序列 $c_g$。最后解码器 $Dec$ 从 $z_q$ 和 $m_q$ 重构语音信号 $\hat{x}$。

编解码器的结构与 HifiCodec 相似：
+ 编码器由一个 1D 卷积层、4 个卷积模块和 1D 卷积层组成。每个卷积模块包含三个残差单元和一个下采样层，总共下采样 320 倍
+ 解码器采用对称结构，使用转置卷积进行上采样

$Q_1$ 将语音量化为 frame-level token。

### Time-invariant 表征提取和量化

TIRE 使用编码器中第二个卷积模块的输出作为输入来提取 time-invariant 表征 $m$。量化后的表征 $m_q$ 输入解码器的第二个转置卷积模块进行解码。
如上图 b，TIRE
+ 先通过三个 1D 卷积层 + LeakyReLU 层对输入语音特征进行进一步提取
+ 对提取的特征进行时间平均，将 frame-level 特征汇总为 time-invariant 特征
+ 通过全连接层和激活函数得到最终的 time-invariant 表征 $m$

time-invariant 表征 $m$ 的量化使用 GVQ，将 $m$ 分为 8 组，得到 8 个离散 token 作为 time-invariant code。

量化后的 time-invariant 表征 $m_q$ 在时间维度上重复，将 segment-level time-invariant 特征转换为 frame-level 特征。

通过 time-invariant code 表示 time-invariant 信息，剩余的 frame-level code 捕捉更大的时间依赖性，从而减少冗余的 frame-level code。

### Time-invariant 编码一致性损失

TiCodec 用于 TTS 时，先从目标语音提取 time-invariant token，然后使用文本信息预测 frame-level token。为了保持从同一话语的不同段提取的一致性，提出 time-invariant encoding consistency loss（$L_c$）：
+ 训练时，除了对语音段 $seg_1$ 编码外，还随机从同一话语中采样另一个段 $seg_2$
+ 分别经过编码器和 TIRE 的前两个卷积模块，最后进行 stop-gradient 操作
+ 使用余弦相似度损失作为一致性损失 $L_c$，计算两个段的 time-invariant 表征的相似度
$$\mathcal{L}_c = 1 - \cos(TIRE(Enc[:2](x_1)), TIRE(Enc[:2](x_2)))$$
其中 $x_1$ 和 $x_2$ 分别表示语 $seg_1$ 和 $seg_2$ 的波形，$\cos$ 表示余弦相似度。

最终 generator 的训练目标为：
$$\mathcal{L} = \lambda_t L_t + \lambda_f L_f + \lambda_g L_g + \lambda_{feat} L_{feat} + \lambda_{qz} L_{qz} + \lambda_{qm} L_{qm} + \lambda_c L_c$$
其中 $\mathcal{L}_t$ 和 $\mathcal{L}_f$ 分别是时域和频域的重构损失，$\mathcal{L}_g$ 是生成器的对抗损失，$\mathcal{L}_{feat}$ 是特征匹配损失，$\mathcal{L}_{qz}$ 是 frame-level code 的量化损失。$\mathcal{L}_{qm} = ||m - m_q||^2$ 是 time-invariant code 的 commitment loss，$\mathcal{L}_c$ 是 time-invariant encoding consistency loss。$\lambda_t, \lambda_f, \lambda_g, \lambda_{feat}, \lambda_{qz}, \lambda_{qm}$ 和 $\lambda_c$ 是超参数，用于平衡最终损失的各个项。

## 实验（略）
