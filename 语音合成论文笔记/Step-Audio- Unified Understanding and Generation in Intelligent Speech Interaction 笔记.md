> Preprint 2025.02，阶跃星辰

1. 提出 Step-Audio，第一个生产级开源模型，贡献如下：
    1. 130B 参数的统一语音文本多模态模型，实现统一理解和生成
    2. 生成式语音数据引擎，低成本语音克隆
    3. 指令驱动的细粒度控制，支持方言、情感、唱歌和 RAP 的动态调整
    4. 增强认知架构，增强工具调用和角色扮演能力
2. 在 StepEval-Audio-360 基准上，Step-Audio 在人类评估中表现出色；在 LLaMA Question 等开源基准上，平均性能提升 9.3\%

## Introduction

1. 现有的开源模型存在三个限制：
    + 理解和生成过程分离，阻碍端到端系统集成
    + 语音数据采集方法麻烦
    + 韵律特征、方言等难以精确控制
2. 现有的框架有：
    + 级联式框架：将 ASR、LLM 和 TTS 级联，但存在错误传播，且系统复杂
    + 纯端到端：对话质量较差
3. 传统的 TTS 依赖手动标注数据集；且现有的方法难以动态调整语音（如实时调整语速、情感韵律或音乐渲染）；缺乏工具调用能力和上下文感知
4. 提出 Step-Audio，通过四点创新统一理解和生成：

    1. 130B 参数多模态模型，集成理解和生成能力，开源 Step-Audio-Chat 版本
    2. 生成式数据引擎，消除对手动数据采集的依赖，开源 Step-Audio-TTS-3B 模型
    3. 细粒度语音控制，支持多种情感、方言和声乐风格
    4. 增强智能，通过工具调用机制和角色扮演提升复杂任务处理能力
5. 在 LLaMA Question、TrivialQA 和 ComplexBench 等基准上，平均提升 9.3 个点；且超过 GLM-4-Voice 和 Qwen2-Audio 等模型

## 相关工作（略）

## 架构

本文采用 AQTA（音频输入，文本输出）+ TTS 框架，如图：
![](image/Pasted%20image%2020250810152148.png)

采用此结构的原因如下：
+ 缺乏高质量纯语音对话数据，限制语音对话模型的训练效率
+ 输出语音的可控性和定制化：通过 TTS 模块，可以控制音色和音调等参数

本文目标为建立 Step-Audio 实时多模态模型，通过以下模块实现语音理解和合成：
+ 双 codebook tokenization 框架，实现 linguistic（16.7Hz，1024-codebook）和 semantic（25Hz，4096-codebook）以 2:3 时间交错并行
+ 130B 参数 LLM，基于 Step-1 模型，采用音频上下文预训练和 postraining 
+ 混合语音合成，结合 flow matching 和 vocoder，优化实时波形生成
+ VAD 模块提取语音片段

### Tokenizer

提出 双 codebook tokenization 框架：
+ linguistic tokenizer，提取结构化高层表征（如音素和语言特征），采用 Paraformer 编码器输出，量化为 16.7Hz 的离散表征，codebook 大小为 1024
+ semantic tokenizer，编码语义和粗粒度声学特征，采用 CosyVoice tokenizer，量化为 25Hz 的离散表征，codebook 大小为 4096

采用了 token-level 的交错方法集成这两种 tokenization，比例为 2:3，即每两个 linguistic token 与三个 semantic token 配对。

### LLM

基于 Step-1 进行音频持续预训练（audio continual pretraining），以提高 Step-Audio 处理语音信息的能力和语音文本对齐的准确性。

多轮对话中音频 token 和文本 token 长度差异很大，因此历史信息在输入系统前先通过 ASR 转为文本。

### Decoder

Decoder 由 3B 参数的 LLM、flow-matching 模型和 mel-to-wave vocoder 组成，输入文本或音频 token，生成波形。采用 dual-code interleaving 方法训练 decoder，确保 linguistic 和 semantic 特征在生成过程中的集成。

### 实时推理

实时推理流程如图：
![](image/Pasted%20image%2020250810160419.png)

核心为 Controller 模块，管理状态转换、响应生成和子系统协调。子系统包括：
+ VAD：检测用户语音
+ Streaming Audio Tokenizer：实时处理音频
+ Step-Audio LLM 和 Speech Decoder：处理和生成响应
+ Context Manager：保持对话连续性

推理相应生成：系统预生成响应以减少延迟。从 Silence 状态开始，等待用户输入。当 VAD 检测到语音时，进入 UserSpeaking 状态，此时 Streaming Audio Tokenizer 开始将音频转换为 token。如果用户暂停，进入 UserPaused 状态，触发预生成响应。若用户继续说话，则丢弃预生成的响应。一旦确定用户说完，进入 BotReplying 状态，提交最新的预生成响应并输出音频。如果在此期间有新的用户输入，则优先处理新输入，同时保持对话连续性。完成响应后返回 Silence 状态，准备下一次交互。

上下文管理：系统使用文本作为历史上下文（提高性能并支持更长对话）。ASR 异步将用户语音转为文本，得到对话历史。

Streaming Audio Tokenizer：输入音频通过两个并行的 tokenizer，两类 token 以 2:3 的比例合并为一个序列。

## 预训练

### 数据集

多模态预训练数据集包含三类数据：
+ 音频：1.1 万亿 token 的音频续写数据（约 730 万小时）、1130 亿 token 的 TTS 合成语音数据（约 70 万小时）、1050 亿 token 的 ASR 数据（约 65 万小时）、3500 亿 token 的音频文本交替数据（约 200 万小时）
+ 文本：8000 亿 token，包含网页、书籍、代码和专有材料
+ 图像：8000 亿 token 的图像文本配对/交替数据，来源于网页、书籍和专有资源

### 训练细节

Step-Audio 是 Step-Omni 的一部分，分为三个阶段训练：
+ 阶段一：扩展预训练文本模型的词表，添加 5120 个音频 token，引入预训练图像编码器得到 Step-Omni 模型。训练时，文本模型学习率保持在 2e-5。此阶段使用音频、文本和图像数据的比例为 2:1:1，仅使用纯音频续写任务。
+ 阶段二：在阶段一训练 1.2T token 后，加入音频文本交替数据，音频续写数据和音频文本交替数据的比例为 1:1。此阶段音频、文本和图像数据的比例仍为 2:1:1。
+ 阶段三：在阶段二训练 800B token 后，加入 ASR 和 TTS 数据，音频续写数据、音频文本交替数据、ASR 数据和 TTS 数据的比例为 1:1:1:1。此阶段音频、文本和图像数据的比例调整为 4:3:3。

### 训练（略）

### 选择 Tokenizer 进行 Audio Pretraining

选择 tokenizer 时发现：
+ 仅使用 semantic token 训练：next token prediction 的 perplexity 低，语义连贯性好，但音色和韵律差，听觉质量差
+ 仅使用 linguistic token 训练：音频不错，但 next token prediction 的 perplexity 很高，语义连贯性差
+ 交错的 semantic token 和 linguistic token 训练：semantic token 保证语义连贯性，linguistic token 保证听觉质量

## 后训练

### TTS

特定语言数据、方言数据、说话风格、情感数据和副语言数据非常稀缺。因此提出合成数据驱动的 TTS 框架，包含三部分：
+ 使用 Step-2 LLM 生成多样化文本
+ 选择包含音频 token 冷却机制的预训练 Step-Audio checkpoint，直接生成特定说话人、语言和方言的音频数据
+ 通过微调上述 checkpoint 开发 Audio-Edit Model，生成细腻的情感表达和多样的说话风格

### AQTA

对于 AQTA 任务采用 RLHF，得到 Step-Audio-Chat 模型，如图：
![](image/Pasted%20image%2020250810202314.png)

## 评估（略）
