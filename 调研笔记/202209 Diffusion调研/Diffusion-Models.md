# Diffusion 模型的方法和应用调研
原论文 [[Diffusion Models- A Comprehensive Survey of Methods and Applications.pdf]]

[Paperwithcode 网址](https://paperswithcode.com/paper/diffusion-models-a-comprehensive-survey-of)

1. Diffusion 可以看成是 VAE 的级联，对应于VAE的编码和解码；Diffusion 也可以看成是 随机微分方程的离散化。
2. 可以将 Diffusion 模型分成三类：
    + sampling-acceleration enhancement
    + likelihood-maximization enhancement
    + generalization-ability enhancement 
3. 其他五种生成模型：
    + VAE
    + GAN
    + normalizing flow
    + autoregressive
    + energy-based models
通过组合这些模型和 Diffusion 模型，可以得到更强大的模型
4. Diffusion 模型应用：
    + cv
    + nlp
    + waveform signal processing
    + multi-modal modeling
    + molecular graph generation（分子图生成）
    + time series modeling（时间序列建模）
    + dversarial purification（对抗降噪）

![](overview.png)

## 预备知识
生成模型一个中心问题就是，模型概率分布的灵活度和复杂度之间的权衡。


Diffusion 模型的基本思想是通过正向过程扰乱数据分布中的结构，然后通过学习反向扩散过程来恢复结构，从而产生高度灵活和易于处理的生成模型。

一共有两种 Diffusion 模型，Denoising Diffusion Probabilistic 模型（更多细节见笔记 [[ddpm]]） 和 Score-based Generative 模型。
![](2022-09-16-11-24-15.png)

### Denoising Diffusion Probabilistic 模型（DDPM）
1. 包含两个马尔科夫链，采用变分推理来生成指定分布的样本
2. forward：从已有的数据分布开始，逐步增加噪声直到分布收敛到某个先验分布（如高斯分布）
3. reverse：从某个先验分布开始，采用带参数的高斯转化核，逐渐恢复出原始数据。
4. 数学角度看，给定目标数据分布 $\mathbf{x}_0 \sim q\left(\mathbf{x}_0\right)$，**forward** 过程为：
    $$\begin{aligned} q\left(\mathbf{x}_1, \ldots, \mathbf{x}_T \mid \mathbf{x}_0\right) &=\prod_{t=1}^T q\left(\mathbf{x}_t \mid \mathbf{x}_{t-1}\right) \\ q\left(\mathbf{x}_t \mid \mathbf{x}_{t-1}\right) &=\mathcal{N}\left(\mathbf{x}_t ; \sqrt{1-\beta_t} \mathbf{x}_{t-1}, \beta_t \mathbf{I}\right) \end{aligned}$$
其中，$\beta_{t} \in (0,1)$ 为方差系数。当 $\alpha_t=1-\beta_t$ 且 $\bar{\alpha}_t=\prod_{s=0}^t \alpha_s$ 时，有：
    $$\begin{aligned} q\left(\mathbf{x}_t \mid \mathbf{x}_0\right) &=\mathcal{N}\left(\mathbf{x}_t ; \sqrt{\bar{\alpha}_t} \mathbf{x}_0,\left(1-\bar{\alpha}_t\right) \mathbf{I}\right) \\ \mathbf{x}_t &=\sqrt{\bar{\alpha}_t} \mathbf{x}_0+\sqrt{1-\bar{\alpha}_t} \epsilon \end{aligned}$$
最终当 $\bar{\alpha}_t$ 接近 $0$ 时，$p(\mathbf{x}_T)$ 和高斯分布无二。
**reverse** 过程为，从 $p\left(\mathbf{x}_T\right)=\mathcal{N}\left(\mathbf{x}_T ; \mathbf{0}, \mathbf{I}\right)$ 开始，参数化反向过程如下：
    $$\begin{aligned} p_\theta\left(\mathbf{x}_{0: T}\right) &=p\left(\mathbf{x}_T\right) \prod_{t=1}^T p_\theta\left(\mathbf{x}_{t-1} \mid \mathbf{x}_t\right) \\ p_\theta\left(\mathbf{x}_{t-1} \mid \mathbf{x}_t\right) &=\mathcal{N}\left(\mathbf{x}_{t-1} ; \mu_\theta\left(\mathbf{x}_t, t\right), \Sigma_\theta\left(\mathbf{x}_t, t\right)\right) \end{aligned}$$
训练过程可以通过优化负的变分下界来实现，通过一系列的推到，最终的优化目标函数可以写为：
    $$\mathbb{E}_{t \sim \mathcal{U}(0, T), \mathbf{x}_0 \sim q\left(\mathbf{x}_0\right), \boldsymbol{\epsilon} \sim \mathcal{N}(0, \mathrm{I})}\left[\lambda(t)\left\|\boldsymbol{\epsilon}-\boldsymbol{\epsilon}_\theta\left(\mathbf{x}_t, t\right)\right\|^2\right]$$
简单来说，就是优化预测噪声和真实噪声之间的L2距离。

### Score-based Generative 模型 
另一方面，可以把 Diffusion 模型看成 score-based generative model 的离散化。这种模型通过构造 stochastic differential equation（随机差分方程，SDE）将数据打乱到一个已知的先验分布。同时一个对应的反向SDE可以反转这个过程。**forward** 过程为以下SDE的解：
    $$d\mathbf{x} = \mathbf{f}(x,t)dt + g(t)d\mathbf{w}$$
其中，$\mathbf{x}_T$ 为打乱后的分布。那么 DDPM 的 forward 过程能够看成SDE前向过程的离散化：
    $$d\mathbf{x} = -\frac 12 \beta(t)\mathbf{x}dt + \sqrt{\beta{(t)}}d\mathbf{w}$$
最后，为了从已知先验生成数据，**reverse** 过程为：
$$d \mathbf{x}=\mathbf{f}(\mathbf{x}, t)-g(t)^2 \nabla_{\mathbf{x}} \log q_t(\mathbf{x}) d t+g(t) d \overline{\mathbf{w}}$$

经典的 Diffusion 模型有三个主要的缺点：
+ 采样过程效率低
+ 似然估计次优
+ 数据泛化能力差
 
增强提高后的 Diffusion 模型可以分为三类：
1. 采样加速
2. 似然最大化
3. 数据泛化


### 采样加速增强（SAMPLING-ACCELERATION ENHANCEMENT）的 Diffusion 模型
现有的方法采样耗时高，以下是一些加速采样方法。

#### Discretization Optimization 离散优化
通过减少采样步是一个可行的方法，但是会影响模型的性能；可以通过优化离散化，如通过求解随机微分方程或常微分方程来减少采样步骤。
1. SGM：以forward SDE相同的方式离散化反向时间SDE。
2. ODE：不需要进行高斯采样，从而提高采样效率

#### Non-Markovian Process 非马尔可夫过程
马尔可夫仅依赖于最近的一个step进行预测，这限制了之前信息的利用。

DDIM 拓展了原始的DDPM来实现非马尔可夫过程。其forward 过程为：
$$q\left(\mathbf{x}_1, \ldots, \mathbf{x}_T \mid \mathbf{x}_0\right)=\prod_{t=1}^T q\left(\mathbf{x}_t \mid \mathbf{x}_{t-1}, \mathbf{x}_0\right)$$
$$q_\sigma\left(\mathbf{x}_{t-1} \mid \mathbf{x}_t, \mathbf{x}_0\right)=\mathcal{N}\left(\mathbf{x}_{t-1} \mid \tilde{\mu}_t\left(\mathbf{x}_t, \mathbf{x}_0\right), \sigma_t^2 \mathbf{I}\right)$$
$$\tilde{\mu}_t\left(\mathbf{x}_t, \mathbf{x}_0\right)=\sqrt{\bar{\alpha}_{t-1}} \mathbf{x}_0+\sqrt{\bar{\beta}_{n-1}-\sigma_t^2} \cdot \frac{\mathbf{x}_n-\sqrt{\alpha_t} \mathbf{x}_0}{\sqrt{\bar{\beta}_t}}$$

DDPM可以看成是DDIM的一个特殊形式。

#### Partial Sampling 部分采样
通过仅使用reverse过程中的部分step来生成样本，以质量换取采样速度。


### 最大似然增强（LIKELIHOOD-MAXIMIZATION ENHANCEMENT）的 Diffusion 模型
DDPM 相比于其他似然模型没有什么优势，通过设计和分析变分下界来增强最大似然，主要有以下三种方法。

#### Noise Schedule Optimization 噪声调度优化
通过优化噪声引入来实现增强的对数似然。如采取以下调度器：
    $$\bar{\alpha}_t=\frac{h(t)}{h(0)}, h(t)=\cos \left(\frac{t / T+m}{1+m} \cdot \frac{\pi}{2}\right)^2$$
同时使用 $\beta_t=1-\frac{\bar{\alpha}_t}{\bar{\alpha}_{t-1}}$ 来计算 forward 方差。由于 $\bar{\alpha}_t$ 平滑改变，reverse 过程能够更好的恢复原始的数据结构。

VDM 表明数据 $\mathbf{x}$ 的变分下界可以简化为 Diffusion 数据的信噪比的简短表达，同时表明，在无限深的设置中，VLB 完全由端点的噪声调度决定，而与端点之间的噪声调度无关，因此可以通过优化端点噪声调度以减少方差。

#### Learnable Reverse Variance 可学习的反向方差

通过优化 reverse 过程的方差的估计可以减少拟合误差，从而最大化VLB和对数似然。通过设置 reverse 方差为：
    $$\Sigma_\theta\left(\mathbf{x}_t, t\right)=\exp \left(v \cdot \log \beta_t+(1-v) \cdot \log \tilde{\beta}_t\right)$$
可以减少采样的时间步。

Analytic-DPM 表明了VLB 的最佳 reverse 方差为：
    $$\Sigma_\theta\left(\mathbf{x}_t, t\right)=\sigma_t^2+\left(\sqrt{\frac{\bar{\beta}_t}{\alpha_t}}-\sqrt{\bar{\beta}_{t-1}-\sigma_t^2}\right)^2 \cdot\left(1-\bar{\beta}_t \mathbb{E}_{q_t\left(\mathbf{x}_t\right)} \frac{\left\|\nabla_{\mathbf{x}_t} \log q_t\left(\mathbf{x}_t\right)\right\|^2}{d}\right) $$

#### Objectives Designing 目标函数设计
完全看不懂。。。

### 数据泛化增强（DATA-GENERALIZATION ENHANCEMENT）的 Diffusion 模型

Diffusion 模型假设数据支持欧几里得空间，即具有平面几何形状的流形。添加高斯噪声将不可避免地将数据转换为连续的状态空间。下面给出两种数据泛化方法。

#### Feature Space Unification 特征空间统一
通过将数据编码到连续的隐空间中，再应用 Diffusion 模型来统一特征空间。LSGM [在基于分数的生成模型中推导出 ELBO 与分数匹配目标函数的联系，从而实现高效的训练和采样。

#### 数据相关的转换核 Data-Dependent Transition Kernels
通过设计适合于数据类型的转换内核来直接 diffuse 数据：
    $$q\left(\mathbf{x}_t \mid \mathbf{x}_{t-1}\right)=\mathbf{v}^{\top}\left(\mathbf{x}_t\right) \mathbf{Q}_t \mathbf{v}\left(\mathbf{x}_{t-1}\right)$$


### Diffusion 模型与其他生成模型之间的联系和结合
生成模型可以用于图像、语音、音乐生成、半监督学习、对抗性识别、模仿学习、强化学习等多个领域，下面介绍了五种生成模型及其和 Diffusion 模型的结合。

#### VAE
VAE 的损失函数（传说中的 ELBO）计算为：
    $$\begin{aligned} L_{\mathrm{VAE}}(\theta, \phi) &=-\log p_\theta(\mathbf{x})+D_{\mathrm{KL}}\left(q_\phi(\mathbf{z} \mid \mathbf{x}) \| p_\theta(\mathbf{z} \mid \mathbf{x})\right) \\ &=-\mathbb{E}_{\mathbf{z} \sim q_\phi(\mathbf{z} \mid \mathbf{x})} \log p_\theta(\mathbf{x} \mid \mathbf{z})+D_{\mathrm{KL}}\left(q_\phi(\mathbf{z} \mid \mathbf{x}) \| p_\theta(\mathbf{z})\right) \\ \theta^*, \phi^* &=\arg \min _{\theta, \phi} L_{\mathrm{VAE}} \end{aligned}$$
其中，$p_\theta(\mathbf{z}|\mathbf{x})$ 是 encoder，$q_\phi(\mathbf{x}|\mathbf{z})$ 是 decoder，通过梯度下降进行优化。

DDPM 可以被看成是固定编码器的分层马尔可夫VAE。 forward 过程代表编码器，reverse 过程代表解码器，且 latent variable 和数据样本的维度一样。

每一步编码过程被建模成 $p\left(\mathbf{x}_t \mid \mathbf{x}_{t-1}\right)$，每一步生成过程则被建模成 $q\left(\mathbf{x}_{t-1} \mid \mathbf{x}_t\right)$，和VAE不同的是，DDPM 的重点在于生成，因此其编码阶段为一个简单的正太分布 $p\left(\boldsymbol{x}_t \mid \boldsymbol{x}_{t-1}\right)=\mathcal{N}\left(\boldsymbol{x}_t ; \alpha_t \boldsymbol{x}_{t-1}, \beta_t^2 \boldsymbol{I}\right)$（VAE是通过一个未知的神经网络来拟合函数），而解码（或者说生成）阶段则被建模成 **均值向量可学习的正态分布** $\mathcal{N}\left(\boldsymbol{x}_{t-1} ; \boldsymbol{\mu}\left(\boldsymbol{x}_t\right), \sigma_t^2 \boldsymbol{I}\right)$，其中 $\mu(\cdot)$ 为待学习的用神经网络拟合求解的均值（注：上述 $\alpha_t, \beta_t$ 的定义和原论文不一样）。

通过和VAE相似的ELBO推导计算，最终的优化目标函数为：
    $$\frac{\beta_t^2}{\alpha_t^2 \sigma_t^2} \mathbb{E}_{\bar{\varepsilon}_{t-1}, \varepsilon_t \sim \mathcal{N}(\mathbf{0}, \boldsymbol{I}), \boldsymbol{x}_0 \sim \tilde{p}\left(\boldsymbol{x}_0\right)}\left[\left\|\varepsilon_t-\boldsymbol{\epsilon}_{\boldsymbol{\theta}}\left(\bar{\alpha}_t \boldsymbol{x}_0+\alpha_t \bar{\beta}_{t-1} \overline{\boldsymbol{\varepsilon}}_{t-1}+\beta_t \boldsymbol{\varepsilon}_t, t\right)\right\|^2\right]$$
降低方差之后，得到的结果和利用[“拆楼-建楼”模型](https://kexue.fm/archives/9119)推出的DDPM一致：
    $$\frac{\beta_t^4}{\bar{\beta}_t^2 \alpha_t^2 \sigma_t^2} \mathbb{E}_{\boldsymbol{\varepsilon} \sim \mathcal{N}(\boldsymbol{0}, \boldsymbol{I}), \boldsymbol{x}_0 \sim \bar{p}\left(\boldsymbol{x}_0\right)}\left[\left\|\varepsilon-\frac{\bar{\beta}_t}{\beta_t} \boldsymbol{\epsilon}_{\boldsymbol{\theta}}\left(\bar{\alpha}_t \boldsymbol{x}_0+\bar{\beta}_t \varepsilon, t\right)\right\|^2\right]$$
#### GAN
GAN 可以被看成是一个 minmax 的优化问题，其目标函数如下：
    $$\min _G \max _D \mathbb{E}_{\mathbf{x} \sim p_{\text {data }}(\mathbf{x})}[\log D(\mathbf{x})]+\mathbb{E}_{\mathbf{z} \sim p_{\mathbf{z}}(\mathbf{z})}[\log (1-D(G(\mathbf{z})))]$$
GAN 的优化目标是实现纳什均衡。

GAN 的主要问题在于训练的不稳定性。通过利用 Diffusion 模型确定的自适应噪声调度器向判别去注入噪声，可以缓解这一问题。同时，可以通过使用 C-GAN 对每个去噪步骤进行建模，从而允许更大的步长以提高模型的采样速度。

#### Normalizing Flow
Normalizing Flow 可以进行精确的密度估计，可以对高维数据进行建模。在离散时间条件下，Normalizing Flow 从数据　$\mathbf{x}$ 到隐变量 $\mathbf{z}$ 的映射是一系列双射函数的组合 $F=F_N \circ F_{N-1} \circ \ldots \circ F_1$，这个过程中，数据串　$\left\{\mathbf{x}_1, \mathbf{x}_2, \ldots \mathbf{x}_N\right\}$　满足：
    $$\mathbf{x}_i=F_i\left(\mathbf{x}_{i-1}, \theta\right), \mathbf{x}_{i-1}=F_i^{-1}\left(\mathbf{x}_i, \theta\right)$$
DiffFlow 通过结合流模型和 Diffusion 模型，综合了两者的优点。

#### Autoregressive Models
自回归模型 (ARM) 是使用概率链规则将数据的联合分布分解为条件分布的乘积：
    $$\log p\left(\mathbf{x}_{1: T}\right)=\sum_{t=1}^T \log p\left(x_t \mid \mathbf{x}_{<t}\right)$$
自回归模型的采样也是一个连续的过程，因此在高维数据时可能非常慢。自回归 Diffusion 模型可以学习生成任意数据的顺序。

#### Energy-based Models
基于能量的模型（EBM）可以看成是判别器的一种生成版本，可以从未标记的数据中进行学习。$\mathrm{x} \sim p_{\text {data }}(\mathrm{x})$ 表示训练样本分布，$p_\theta(\mathbf{x})$ 表示用于拟合数据分布的模型的概率密度函数，基于能量的模型定义如下：
    $$p_\theta(\mathbf{x})=\frac{1}{Z_\theta} \exp \left(f_\theta(\mathbf{x})\right)$$
其中 $Z_\theta=\int \exp \left(f_\theta(\mathbf{x})\right) d \mathbf{x}$ 为配分函数，高维难解。

EBM的挑战在于，通过最大似然来学习EBM需要用MCMC的方法从模型中生成样本，非常耗时耗力。同时会导致学习的能量势不稳定。

### Diffusion 模型的应用

#### Computer Vision


任务：
+ 图像超分辨率：从低精度的图片生成高精度的图片
+ 图像修补：修复原图像的丢失或者破坏区域

模型： 
+ Super Resolution Diffusion ： 基于 Diffusion 模型，优化变分下界，通过调节噪声来逐步生成高精度的图片。
+ Super-Resolution via Repeated Refinement ： 通过随机迭代降噪过程实现超精度。
+ LDM ： 提出  latent diffusion 模型，提高训练和采样效率而不减少模型质量。
+ RePaint ： 通过重采样迭代设计了一个改进的降噪策略。
+ Palette ： 基于条件 Diffusion 模型提出了一个统一的框架，用于四种图像生成任务。
+ Cascaded Diffusion Models ： 级联多 Diffusion 模型，逐步提高精度。
+ Multi-Speed Diffusion ： 提出条件多速率diffusive estimator。结合 conditional score estimation 

任务：
+ 语义分割：将图像中属于相同类别的进行聚合

模型：
> 有研究表明，DDPM 可以捕获高维的语义信息
+ Decoder Denoising Pretraining : 采用有监督学习的方法初始化encoder，通过 denoising 目标函数预训练decoder

任务：
+ 异常检测：生成模型在异常检测中是一个很强大的模型

模型：
+ AnoDDPM ： 利用 DDPM 破坏输入图像，并重构一个正常的相似图像
+ DDPM-CD ： 用预先训练的DDPM并应用 Diffusion 模型解码器的多尺度表示来进行变化检测。

任务：
+ 点云补全和生成
模型：略

任务：
+ 视频生成
模型：
> 高质量的视频生成很具有挑战性
+ Flexible Diffusion Model
+ Residual Video Diffusion
+ Video Diffusion Model

#### NLP
Diffusion + 文本生成
Diffusion + LM

#### Waveform Signal Processing
WaveGrad 引入了用于估计数据密度梯度的波形生成条件模型。
DiffWave 通过优化数据分布的变分界来进行训练。，可以进行有条件或无条件的波形生成。

#### Multi-Modal Learning（多模态学习）

任务：
+ Text-to-Image Generation：文本生成图像
模型：
+ Blended diffusion 
+ unCLIP
+ Imagen
+ GLIDE
+ VQ-Diffusion

任务：
+ Text-to-Audio Generation：文本生成音频
模型：
+ Grad-TTS ： 基于 score 的解码器和 Diffusion 模型，逐渐转换编码器预测的噪声。
+ Grad-TTS2 ： 自适应改进一代。
+ DiffSound ： 提出基于离散 Diffusion 模型的非自回归解码器，在每个step中预测所有的mel谱 token，在接下来的步骤中进行细化。
+ EdiTTS ： 利用基于 score 的TTS模型来细化Mel谱。
+ ProDiff ： 直接预测干净的数据来参数化 DDPM。

#### Molecular Graph Modeling（略）

#### Time Series Modeling（时间序列建模）
任务：
+ Time Series Imputation（时间序列插补）
模型：
+ 之前方法使用自回归模型及其变形进行处理
+ Diffusion 模型：
  + CSDI ： 基于 score 的 Diffusion 模型，采用自监督形式来优化模型
  + CSDE ： 提出新的概率框架，使用神经网络控制随机微分方程对随机动力学进行建模
  + SSSD :  集成条件 Diffusion 模型和结构化状态空间模型，用以特别捕获时间序列中的长期依赖关系

任务：
+ Time Series Forecasting （时间序列预测）
模型：
+ TimeGrad ： 预测多元概率的时间序列自回归模型，利用了与分数匹配和基于能量的方法密切相关的DDPM。

#### Adversarial Purification（对抗性提纯，对抗样本降噪）
使用生成模型来消除对抗性扰动的一类防御方法。使用提纯模型将受攻击的图像提纯为”干净“的图像。

DiffPure 在 forward 过程之后用少量的噪声对其进行 Diffusion，然后通过 reverse 过程恢复干净的图片。

Adaptive Denoising Purification 表明了使用去噪分数匹配 (DSM) 训练的基于能量的模型（EBM）只需几个步骤即可有效地提纯受攻击的图像

Projected Gradient Descent 提出一种新颖的基于 Diffusion 的随机预处理鲁棒性。

### 未来方向
1. 重新检查实际可用的条件
2. 从离散转到连续时间的 Diffusion
3. 新的生成过程
4. 推广到复杂场景和更多的研究领域
