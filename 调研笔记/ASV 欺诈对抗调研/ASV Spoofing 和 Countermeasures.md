# Spoofing and countermeasures for speaker verification: A survey 综述
> 2015 年的一篇论文，早于 ASVspoof 2015 比赛。

1. 本文回顾了过去在模仿、重放、语音合成和语音转换方面的欺骗攻击，以及最近（2015年以前）的反欺骗对策。
2. 未来研究应该解决一下问题：
	1. 标准数据集的缺乏
	2. 现有对策对特定、已知的欺骗的过拟合


## Introduction
1. 在麦克风和传输端的攻击危害性最大
2. 利用短时谱特征的ASV系统很容易受到欺骗
3. 解决欺骗问题的两个方法：
	1. （继续传统的）改进基本性能
	2. 设计特定的或者通用的反欺骗对策
4. 本文比较了四种不同的攻击的漏洞，回顾针对每种攻击的反欺骗对策，最后讨论未来方向

## ASV
1. ASV 的任务：根据语音样本接受或者拒绝一个声明的身份。
2. ASV的两种类型：
	1. 文本依赖，通常采用固定的短语或提示语；更适合认证场景
	2. 文本独立：任意话语，甚至可能不同的语言；适用于来电验证、监视等。

下面介绍ASV的最新技术。
### 特征提取
1. 语言信号的三种信息：
	1. 音色——短时谱特征，如 MFCC、LPCC、PLP等
	2. 韵律——韵律特征，如音高、能量、持续时间
	3. 内容——语言内容特征，从词典中提取，语音识别等

### 说话人建模和分类的发展（文本无关）

1. 采用对数似然比来计算得分：$\ell=\log \frac{p\left(\mathbf{X} \mid H_0\right)}{p\left(\mathbf{X} \mid H_1\right)}$ ，根据某个阈值来拒绝或者接受假设 $H$。其中的 $\mathbf{X}$ 可以是 MFCC，也可以是 i-vector等等特征。
2. GMM+UBM 为行业标准
3. GMM supervectors + SVM 用于生成模型的鉴别训练
4. 联合因子分析(JFA)通过结合不同的说话人和通道子空间模型来改善 ASV 的性能
5. JFA 改进变成 i-vector，同时采用PLDA进行后端处理
6. 对于文本相关的系统，还可以对语言内容进行建模

### 系统融合
1. 多个独立训练的识别器分别捕获语言信号的不同方面
2. 一种简单的方法为，对基本的分类器进行加权求和，权重值看作是待优化的参数

## 说话人验证对欺骗的脆弱性
### 可能的攻击点
一个典型的ASV包括两个过程，离线注册（offline enrolment）和运行验证（runtime verification）。
1. 在 offline enrollment 中，利用音频特征对目标说话人进行模型训练
2. 在 runtime verification 中，说话人声明一个身份，给出其音频样本，提取相关的特征与模型进行比较，以确定身份是否匹配。

实际过程中，样本和两个模型进行对比，一个是假设的说话人，另一个代表可能的假设，分类器确定一个匹配分数，该分数表示样本与两个模型中的每一个的相对相似度。最后，决策模块使用相对得分(通常是对数似然比)来接受或拒绝身份声明。

这个过程中的漏洞点可以分为一下两个部分：
![[Pasted image 20221005145924.png]]
1. 直接攻击：可以应用在麦克风级别以及传输级别（图中的1和2）。例子包括模仿他人或在麦克风前播放预先录制或合成的语音信号
2. 间接攻击：在 ASV 系统内部进行的（图中3-8）。间接攻击通常需要系统级访问，例如干扰特征提取(3，4)、模型(5，6)或评分和决策逻辑计算(7，8)的攻击。

由于直接攻击不需要系统级别的访问，他们是最容易实现的攻击，也是最大的威胁。

在麦克风端，通常用模仿和重放攻击；在传输级别端，通常使用语音合成和语音转换进行攻击。

### 潜在的攻击漏洞

#### 在特征提取端
1. 预录制的语音样本和重放攻击可以完美的反映原始说话人的频谱特征
2. 使用短时谱特征时，最先进的语音合成和语音转换都可以伪造出特定说话人的短时谱特征
3. 韵律特征也可以通过模仿和训练特定的模型来实现
4. 语音内容更容易被模拟了（重放就是直接播放录制好的音频）

#### 在说话人建模端
1. 大多数说话人建模都基于GMM，缺乏时间序列信息建模，而典型的语音合成和转换系统也假设特征独立，可以实现有效的攻击
2. 多生物特性检测中，在单个显著加权子系统的欺骗特别有效的情况下，在评分融合设置下仅欺骗一种模式(或子系统)。传统的融合技术可能无法显著提高的欺骗鲁棒性，除非与专用的欺骗对策相结合。


## 评估方案
本节讨论数据库设计和评估指标

### 数据库设计
过去研究中的欺骗攻击的通用框架如下：
![[Pasted image 20221005153705.png]]
欺骗攻击的三种输入：
+ 真实语音
+ zero-effect impostor 语音
> An impostor attempt is classed as ‘‘zero-effort’’ if the individual submits his/her own biometric feature as if he/she were attempting successful verification against his/her own template, but the comparison is made against the template of another user. 
+ 虚假语音

这一段没看懂。。。

### 评估指标
ASV 系统评估需要两种测试，目标测试，说话人和声称的身份匹配；冒名顶替测试，说话人和声称的身份不匹配。ASV 系统则一共有四种可能的情况：
![[Pasted image 20221005155126.png]]
有两种可能的正确结果和两种可能的错误结果，即错误接受(虚警)和错误拒绝(漏警)。从多个独立测试(试验)获得的统计数据被用来估计错误接受率(FAR)和错误拒绝率(FRR)。FAR 和 FRR 是互补的，对于一个可变阈值的系统，一个只能以增加另一个为代价来减少。在实践中，所有系统参数都被优化以最小化 FAR 和 FRR 之间的平衡，这通常以等误差率(EER)来衡量。

在欺骗场景中，攻击者试图使系统结果偏向于接受错误的身份声明。欺骗攻击也会增加 FAR（固定决策阈值）。

而反欺骗对策应该尽可能减少 FAR，而在真正的音频输入时不增加 FRR（简单来说，就是提高对虚假语音的检测成功率，但是不减少对真实语音的误判率），但是一个实际的系统通常不可避免的导致一些错误接受和错误拒绝。

除了 EER，检测成本函数（DCF）表示使用目标事件和非目标事件的先验概率在 FAR 和 FRR 之间的权衡，DCF 在 ASV 系统中得到了广泛应用，但是在 欺骗和对抗 领域还没有被广泛应用。

## 欺骗和反欺骗（对抗）措施
以往的 ASV 系统几乎没有能力抵抗欺骗攻击，本节回顾了这些事实和开发反欺骗的一些工作。在模拟、重放、语音合成、语音识别中的攻击考虑三个不同的方面：
+ 欺骗攻击的实用性
+ 系统受攻击时的脆弱性
+ 实验真实数据集的设计
聚焦于，
1. 防止某种特定攻击的对策的有效性
2. 在防止各种不同攻击的对策的泛化性

### 语音模拟
> 在没有辅助科技的情况下人工模仿目标说话者的声音音色和韵律。
#### 欺骗攻击
1. 模拟语音可以将 FAR 率从接近0% 提高到10% 到60% 之间
2. 研究表明经典的 GMM-UBM 和最先进的 i-Vector 系统对模拟攻击的很脆弱
3. 声学-语音研究表明，尽管模仿者在模仿长期的韵律 F0模式和说话速率方面往往是有效的，但在模仿共振峰和其他光谱特征方面可能效果较差
4. 由于模仿被认为主要涉及模仿韵律和风格，所以它可能被认为比当今最先进的 ASV 系统更有效地欺骗人类听众
5. 研究表明，一些冒名顶替的演讲者天生就有可能与其他演讲者混淆。类似地，某些目标说话人可能比其他人更容易被模仿。
6. ASV 系统本身甚至是声学特征单独可以被用来识别“相似”的说话人
7. 先前的模拟攻击有一定程度的不一致性，仍然需要进一步的研究

#### 反欺骗对策
1. 模仿者较少使用模仿的声音，因此在伪装下表现出更大的(夸张的)声学参数变化，通过量化声学变化来实现，该过程需要元音分割、强制对齐和手动纠错等。
2. 研究新的指标来预测模拟特定目标声音的难易程度可能是有益的


### 重放
> 使用来自真正的目标说话人的样本的预录制音频来攻击ASV系统

由于录制成本低、录制方便，重放攻击是最容易实施的攻击，威胁也最大。

#### 欺骗攻击
1. 研究表明，重放攻击会引起 EER 和 FAR 的显著增加
2. 研究给出，一个文本无关的 GMM-UBM 系统在遭受重播攻击时的 EER 为40.17% 
3. 使用频谱特征的 ASV 系统容易受到重放攻击

#### 反欺骗对策
1. Shang and Stevenson 首次提出了一种使用固定密码短语的文本依赖 ASV 系统下的重播检测
2. Villalba and Lleida 等人提出了一种基于光谱比和调制指数的替代对策，其动机源于噪声和混响的增加，这是由于重播远场记录而产生的。
3. Wang 提出一种基于信道噪声检测的重放攻击对策。因为原始录音只包含来自 ASV 系统的录音设备的通道噪声，而重放攻击会产生由录音设备和用于重放的扬声器引入的额外通道噪声
4. 或采用 说话人识别+人脸识别的多模态方法，结合视觉信号检测重放攻击

### 语音合成
语音合成的发展、原理略。

#### 欺骗攻击
大多数的语音合成攻击假设攻击在传输级中。
1. 基于HMM的合成语音很容易受到攻击，当用真实语音进行测试时，ASV 系统的 FAR 为0% ，FRR 为7% 。当受到合成语音的欺骗攻击时，FAR 增加到70% 以上。
2. 另一个实验中，使用最先进的基于 HMM 的语音合成器，GMM-UBM 和 SVM 系统的 FAR 分别从0.28% 和0% 上升到86% 和81% 
3. 与模仿和重放攻击的研究不同，语音合成攻击通常使用相对大规模的数据集，通常是高质量的干净语音。所有过去的工作都证实，语音合成攻击能够显著提高所有测试过的 ASV 系统(包括那些处于最先进水平的 ASV 系统)的 FAR。

#### 反欺骗对策
1. 大多数检测合成语音的方法依赖于处理特定于特定合成算法的伪影。
2. 基于合成语音语音参数的动态变化趋于小于自然语音语音参数的观察，因此有研究了使用帧内差异作为区分特征
3. 高阶梅尔倒谱系数(MCEP)也被用来检测基于 HMM 的合成系统产生的合成语音。在 HMM 模型参数训练过程中，反映谱包络细节的高阶倒谱系数趋于平滑。因此，合成语音的高阶倒谱分量比自然语音的方差小
4. 人类听觉系统对相位相对不敏感，声码器通常不重建语音的相位信息。这种简化导致了人类和合成语音之间的相位谱的差异，这种差异可以用于区分
5. 基于单位选择和统计参数语音合成在韵律建模方面的很困难：统计参数语音合成方法生成的 f0往往过于平滑，单位选择方法经常在语音单位的拼接点出现“ f0跳跃”

### 语音转换
语音转换系统的输入是自然语音信号。通常，语音转换涉及谱映射和韵律转换。谱映射关系到声音的音色，而韵律转换关系到韵律特征，如基频和持续时间。

谱映射有三种方法：
+ 统计参数法
+ 频率扭曲法
+ 单位选择法

语音转换技术对使用频谱技术、韵律特性的 ASV 系统构成威胁。语音转换的发展、方法等略。

#### 欺骗攻击
1. 一个名为YOHO的语料库评估了GMM-UBM ASV系统的脆弱性。实验表明，由于语音转换攻击，FAR从略高于1%增加到86%，未来也有很多实验验证了语音转换攻击会显著提高ASV系统的EER
2. 在基于GMM的语音转换攻击中，JFA系统的FAR从3.24%增加到17%以上。面对单位选择转换攻击，最强大的PLDA系统的比例从2.99%增加到40%以上。这些结果是由于真实语音和转换语音的ASV分数分布有相当大的重叠。
3. 有研究表明，语音转换会导致所有的 i-vector, GMM-NAP 和 HMM-NAP三个系统的EER和FAR增加

#### 反欺骗对策
1. 由于合成和转换的某些相似性，一些检测转换语音的工作借鉴了合成语音检测的相关工作
2. 有研究采用余弦归一化相位（cos相位）和修正群延迟相位（MGD相位）特征来区分真实语音和合成语音
3. 基于超矢量的SVM分类器对人工信号攻击具有天然的鲁棒性，可以使用语音水平估计值和动态语音可变性有效地检测语音转换攻击。与自然语言相比，转换后的语言表现出更少的动态变化。

## 其他
关于 欺骗和反欺骗对策的一些讨论。

### 欺骗
关于前面四种欺骗和反欺骗对策的总结如下图：
![[Pasted image 20221006094920.png]]
分别从可达性和有效性两个点进行分级。可达性反映了攻击的容易程度，有效性反映了每次攻击导致的 FAR 增加量，或者说对 ASV 造成的风险等级。表格说明：
1. 无论是在文本无关的系统还是固定短语的文本相关系统中，重放攻击相比于模仿，是一种更容易、更有效的欺骗方法。
2. 目前（2015年）无论是语音合成还是语音转换，都无法用在商用系统中，但其仍然有明确的商业应用，一些语音合成系统已经能够产生与人类语音质量相当的语音，但是其可达性最高（即技术门槛高），在某些攻击中，语音合成和语音转换的欺骗攻击对ASV的性能构成最大的威胁。

### 反欺骗对策
反欺骗对策仍然处于萌芽阶段，被其他生物特征的鉴伪技术甩开一条街。上表也给出了每个欺骗对策的可用性。
1. 由于语音模拟是自然的，因此不存在可以用于检测目的的processing artefacts，因此模拟语音的对策一栏是 ”不存在“
2. 很少有文献给出针对重放攻击的对策，因此重放攻击对策的可用性很低
3. 如果欺骗者已经知道这些对策，则可以很轻松的进行应对，例如，基于相位相关特征的对策可以通过包含自然相位信息来克服

### 通用对策（一般性对策）
上述所有过去的工作都针对特定形式的欺骗，通常利用特定欺骗算法的一些先验知识。然而，在实践中，无论是欺骗的形式还是确切的算法都无法确定。
1. 在关于合成语音欺骗和转换语音欺骗的独立研究中，强调了通用对策的潜力。由于这两种形式的攻击都采用了语音编码技术，因此在这两项研究中，相位信息的使用被证明是检测操纵语音信号的可靠手段
2. Alegre等人基于二进制模式（LBP）对抗措施能有效检测完全不同（无通用声码器）的语音合成和人工信号攻击
3. 研究表明，广义单类分类器在检测合成语音和转换语音方面都是有效的。


## 未来研究
本节讨论了当前的评估协议和指标以及方法中的一些弱点，还讨论了一些开源软件包。

### 大规模数据集
1. 未来的工作将需要更大规模的标准数据集，这些数据具有真实的信道和多样的记录环境，以便能够在现实条件下可靠地比较各种形式攻击的威胁。
2. 过去在反欺骗对策方面的工作都显示或隐式的已知了欺骗的形式，但这不能代表实际场景的实际情况，因此可以认为过去的工作的性能被高估了，此外，大部分工作的语音样本都是在相同或类似声学环境中采集的。因此需要大规模的标准数据集，不仅可以通过实际信道或记录环境的变化来评估对抗性能，而且还可以在缺乏先验知识的情况下或在可变攻击下评估对抗性能。

### 评估指标
1. 目前没有标准评估协议、指标或ASV系统，因此需要在未来定义此类标准
2. 预期性能和可欺骗性曲线（EPSC）为评估具有集成欺骗对策的生物特征识别系统提供了一种替代方法


### 开源软件包
1. ALIZE库和相关工具箱
2. Bob信号处理和机器学习工具箱

### 未来方向
1. 通用对策开发：此类工作可能建立在一类方法的基础上，其中对策只使用自然语言进行训练。评估协议应包括各种不匹配的欺骗技术，从而反映出欺骗攻击可能性质的不确定性。
2. 文本依赖系统：未来的工作应该更多地关注与文本相关的系统，更适合于身份验证场景。
3. 重放攻击：应更加重视效率较低、但更容易实现的重放攻击
4. 声学失配下的对策：在实践中，应考虑到不同的传输通道、附加噪声和其他缺陷，和有可能掩盖对欺骗检测至关重要的处理伪影。因此，未来评估应评估声学退化和通道失配条件下的对策。
5. 多欺骗攻击下的对策：过去的大多数研究只涉及应对独立的欺骗方法。未来的工作应该考虑在攻击者结合几种欺骗技术情况下来提高检测效率。

总的来说，标准数据库、协议和测量指标将成为未来欺骗和泛反欺骗对策研究的重要组成部分。
