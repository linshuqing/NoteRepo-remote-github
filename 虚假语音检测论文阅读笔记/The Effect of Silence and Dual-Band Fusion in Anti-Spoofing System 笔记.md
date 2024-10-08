> InterSpeech 2021，声学所、UCAS

1. 欺诈检测系统在进行 语音活性检测（VAD，也叫语音端点检测）之后性能发生下降。
2. 本文调查了语音开始和结束阶段时静默的影响，发现静默是的差异是反措施判断的基础之一
3. 在VAD操作去除无声片段后，神经网络会丢失这些片段的信息。这可能导致更严重的过拟合
4. 研究发现，特征的高频部分是系统过拟合的主要原因，而低频部分对已知攻击的鲁棒性更强
5. 提出了双频融合反欺骗算法，该算法只需要两个子系统，ASVspoof 2019 LA 下的性能优于除一个外的所有


## Introduction
1. 本文发现，VAD  会加剧过拟合现象，并基于此提出了更鲁棒的算法
2. [[Audio replay attack detection with deep learning frameworks 笔记]] 和 [[STC Antispoofing Systems for the ASVspoof2019 Challenge 笔记]] 使用 LCNN 进行欺诈检测，[[ASSERT- Anti-Spoofing with Squeeze-Excitation and Residual neTworks 笔记]] 和 [[The DKU Replay Detection System for the ASVspoof 2019 Challenge- On Data Augmentation, Feature Representation, Classification, and Fusion 笔记]] 使用 ResNet 进行欺诈检测，[[Siamese Convolutional Neural Network Using Gaussian Probability Feature for Spoofing Speech Detection 笔记]]使用孪生卷积网络进行欺诈检测
3. 但是用神经网络的一个问题就是过拟合，且很少用到子带信息，因此本文研究了在不同频率的子带中基于 ResNet 的反欺诈性能
4. 将频谱分成高频和低频，作为特征输入到 [[SENet]] 中，实验发现过拟合主要是由高频部分引起的，因此提出了基于 SENet 的低频反欺骗算法，并在此基础上提出了计算量小、鲁棒性好的双频融合反欺骗算法。


## 静音和双频融合

### 静音差异
静音差异定义为真实语音和虚假语音在开头和结尾的静音片段之间的差异。

真实语音的静音段和虚假语音差异较大：虚假语音的静音段要么没有静音段，要么静音段没有噪声，要么静音段的噪声异常

> 也就是说，网络很可能学到了这一部分差异，从而导致一定程度的过拟合，所以进行 VAD 之后会导致较差的结果

因此在本文的实验中，通过丢掉数据的开头和结尾以消除静音差异或执行VAD，可以确定对策是否实际提取了有效信息，而不是基于静音段进行判断。

### 双频融合反欺骗算法
[[Subband modeling for spoofing detection in automatic speaker verification 笔记]] 研究了不同子带的影响及其对 PA 检测的重要性，但是还没有研究子带信息对 LA 的性能。

神经波形TTS方法合成的语音与早期声码器方法之间存在明显的差距，尤其是在高频下；训练集中的数据不足可能导致神经网络过于关注高频差异；所以提出了双频带融合算法。

采用的基本特征是频谱图，它被分成两部分：低频部分（0− 4kHz）和高频部分（4− 8kHz）。将频谱图的两部分输入两个SENet进行训练，得到两个反欺骗系统。最后两个子带系统的得分进行线性等权融合。


## 实验设置

### 数据集和指标

ASVspoof 2019 LA 、 EER + t-DCF

### 前端特征
特征：对数功率幅度谱

固定长度为 600 帧，窗口：1728，帧移：130 ，Blackman 窗。最终特征维度为 $865 \times 600$，谱图在连接之间进行左右翻转以避免不连续性。

由于是双频带系统，每个子系统的输入为 $433 \times 600$。

采用了基于端点检测的 VAD 方法。

### 模型架构
本文采用 LCNN 和 SENet 作为分类器。SE block 可以自适应地获取每个特征通道的重要性，并通过为它们分配权重来显式地建模特征通道之间的相关性。基于这种特征，SENet 能够有效地区分真实和虚假语音。

本文使用的 SENet 如下：
![[Pasted image 20221026214837.png]]

### 训练

损失函数：A-softmax
参数 ：β1 = 0.9, β2 = 0.98, ϵ = 10−9 and weigth decay 10−4

对于SENet，学习速率在前1000个预热步骤中线性增加，然后与步骤数的平方根成比例地减小。

SENet 训练 32 epoch ，LCNN 16 个，选择 loss 最小的作为最终评估模型。


## 结果

下图表明，静音段确实影响性能。
![[Pasted image 20221026215141.png]]
结论是，通过使用VAD去除所有静音片段，反欺骗系统可以更加关注语音本身的差异，而不是静音片段。（但是 VAD 并不影响一直欺骗的检测性能）

下表给出，是否进行 VAD、不同频带的 SENet 性能：
![[Pasted image 20221026215512.png]]
总结：
1. 开发集中，单系统已知攻击时，全频带FFT特性与SENet模型表现最佳。
2. 评估集中，低频特征对于评估集中的所有未知攻击方法具有更稳定和更好的性能，而全频带特征和高频特征过拟合

总之，高频特征能够更好地抵抗已知的欺骗攻击方法，但会导致严重的过拟合。低频特征可以有效地检测大多数欺骗方法，但在已知攻击面前的准确性略低。两个子带系统的融合将提高对抗性能。

单系统性能比较：
![[Pasted image 20221026220302.png]]
总结：单系统很强，多系统也不赖。

