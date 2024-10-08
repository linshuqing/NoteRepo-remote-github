> Arxiv, 2022 12.20

1. LLM 严重依赖于人类给出的指令
2. 引入 SELF-INSTRUCT，从语言模型中生成 指令、输入和输出，然后进行剪枝，最后用这些来 fine tune 原始的模型
3. 采用提出的方法训练 GPT-3，能获得 33% 的性能提升

## Introduction
![[Pasted image 20230331152513.png]]
1. 引入 SELF-INSTRUCT，半自动化的过程，使用来自模型本身的指令来 fine tune 模型，整个过程是一个迭代算法，从175个种子任务开始（这一部分是工人的指令，对于每个任务，一条指令和一条输入输出对）：
	1. 第一阶段，根据前面的 prompt 来生成用于新任务的指令
	2. 给定这些指令，framework 同时也会生成对应的输入输出（用于后面有监督的 instruction tuning）
	3. 通过各种措施对这些低质量的重复的指令进行剪枝，然后加入任务池
	4. 重复上面的任务多次直到生成大量的任务
2. 把这个 framework 用在 GPT3 上，最后生成了 52K 的指令，对应于 82K 的输入输出对，把这些数据用于 fine tune GPT-3，结果表明效果优于原始的 33%，几乎到达了 InstructGPT 的性能

## 相关工作（略）

## 方法

### 定义指令数据

生成的指令集 $\left\{I_t\right\}$ ，每个都定义了一个任务 $t$，每个任务有一个或多个输入输出对 $\left(X_t, Y_t\right)$，模型 $M$ 用于在给定输入和指令的情况下生成输出：$M\left(I_t, x\right)=y$，同时这里的 $x$ 允许为空，一个例子是：
+ 指令：write an essay about school safety，输入为 空
+ 指令：write an essay about the following topic，输入为 school safety

### 指令数据的自动生成
包含四个步骤：
+ 指令生成
+ 判断指令是否属于某个任务类
+ instance generation with the input-first or the output-first approach
+ 过滤低质量的数据

#### 指令生成

SELF-instruction基于这样一个发现，即当在上下文中与一些现有指令一起呈现时，可以 prompt 大型预训练语言模型生成新的和新颖的指令。这为我们提供了一种从一小组人工编写的种子指令中增长指令数据的方法。

首先手动创建 175 个任务，每个任务包含 一条指令和一条输入输出对，作为初始任务池。在每个 step 中，从池子选 8 条指令作为 prompt，其中 6 条来自人工创建的，2 条来自之前生成的来提高多样性：![[Pasted image 20230331161325.png]]

#### 分类任务识别

对于分类任务和非分类任务的方法是不一样的，所以需要单独识别出其中的分类任务，这个过程也是用模型自己判断的，用的 prompt 如下：![[Pasted image 20230331161850.png]]

#### 输入输出对生成

给定 instruction 和任务类别，为每条指定生成独立的输入输出对。模型需要根据指令理解具体的任务，同时找出所需的额外的字段。最后生成输出。

有两种方法：
+ Input-first：先生成输入，再生成输出
+ output-first：先生成标签，再生成输入

具体的例子见论文。

#### 过滤和后处理

为了提高多样性，计算指令的重叠度，小于某个阈值才加入到任务池中。同时过滤掉了一写包含特定关键字（如图像、图表等）的指令，当为每个指令生成新的输入输出对时，会过滤掉完全相同的或者输入相同输出不同的那些对。

#### 基于指令 fine tune 模型

创建完前面的指令数据之后，基于这些数据 fine tune 模型，同时为了使模型对不同格式更具鲁棒性，将指令和输入输出对以多种方式进行了组合。