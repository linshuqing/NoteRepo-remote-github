> Preprint 2025.02，阶跃星辰
<!-- 翻译&理解 -->
<!-- Real-time speech interaction, serving as a fundamental interface for human-
machine collaboration, holds immense potential. However, current open-
source models face limitations such as high costs in voice data collection,
weakness in dynamic control, and limited intelligence. To address these
challenges, this paper introduces Step-Audio, the first production-ready
open-source solution. Key contributions include: 1) a 130B-parameter uni-
fied speech-text multi-modal model that achieves unified understanding and
generation, with the Step-Audio-Chat version open-sourced; 2) a generative
speech data engine that establishes an affordable voice cloning framework
and produces the open-sourced lightweight Step-Audio-TTS-3B model
through distillation; 3) an instruction-driven fine control system enabling
dynamic adjustments across dialects, emotions, singing, and RAP; 4) an en-
hanced cognitive architecture augmented with tool calling and role-playing
abilities to manage complex tasks effectively. Based on our new StepEval-
Audio-360 evaluation benchmark, Step-Audio achieves state-of-the-art per-
formance in human evaluations, especially in terms of instruction following.
On open-source benchmarks like LLaMA Question, shows 9.3% average per-
formance improvement, demonstrating our commitment to advancing the de-
velopment of open-source multi-modal language technologies. Our code and
models are available at https://github.com/stepfun-ai/Step-Audio. -->
1. 提出 Step-Audio，第一个生产级开源模型，贡献如下：
    1. 130B 参数的统一语音文本多模态模型，实现统一理解和生成
    2. 生成式语音数据引擎，低成本语音克隆
    3. 指令驱动的细粒度控制，支持方言、情感、唱歌和 RAP 的动态调整
    4. 增强认知架构，增强工具调用和角色扮演能力
2. 在 StepEval-Audio-360 基准上，Step-Audio 在人类评估中表现出色；在 LLaMA Question 等开源基准上，平均性能提升 9.3\%

## Introduction
<!-- The evolution of artificial intelligence toward general-purpose systems has po-
sitioned real-time speech interaction as a critical interface for human-machine
collaboration. While recent multi-modal large language models (LLMs) have
accelerated progress in this domain, open-source communities face persistent
challenges despite breakthroughs in proprietary systems like GPT-4o (Hurst et
al., 2024) and Doubao (bytedance, 2024). Existing open-source models such as
Qwen2-Audio (Chu et al., 2024a), Llama 3 (Dubey et al., 2024) and wavLLM (Hu
et al., 2024) struggle with three fundamental limitations: the separation of under-
standing and generation processes that impedes end-to-end system integration,
dependence on laborious manual speech data acquisition methods that restricts
efficient voice replication, and inadequate precision in regulating prosodic features,
regional dialects, and tool utilization capabilities. These limitations highlight the
urgent demand for deployable frameworks that harmonize streamlined architec-
ture with dual competencies in affective computing (accurate emotion perception
and adjustment) and contextual cognition (situational reasoning and response
formulation). -->
1. 现有的开源模型存在三个限制：
    + 理解和生成过程分离，阻碍端到端系统集成
    + 语音数据采集方法麻烦
    + 韵律特征、方言等难以精确控制
<!-- Current open-source speech systems confront multiple architectural challenges.
The traditional framework employs a cascading approach (Huang et al., 2024)
combining Automatic Speech Recognition (ASR), LLM processing, and Text-to-
Speech (TTS). This framework introduces error propagation through modality
transitions while increasing system complexity. Pure end-to-end approaches,
though conceptually elegant, often sacrifice performance in open-domain dialogue
quality (Zeng et al., 2024). The tension between modular design and fully
integrated systems remains unresolved. Furthermore, traditional text-to-speech
pipelines depend on manually curated datasets, particularly for multilingual and
multidialect scenarios—a process requiring prohibitive human annotation effort.
Existing solutions also lack sophisticated control mechanisms for dynamic speech
adaptation, such as real-time adjustment of speaking rate, emotional prosody,
or musical rendering (e.g., Singing and RAP vocals). Crucially, the absence of
tool invocation capabilities and contextual awareness prevents handling complex
queries like “Retrieve live weather data and report it in Cantonese,” necessitating
manual API integration. -->
2. 现有的框架有：
    + 级联式框架：将 ASR、LLM 和 TTS 级联，但存在错误传播，且系统复杂
    + 纯端到端：对话质量较差
3. 传统的 TTS 依赖手动标注数据集；且现有的方法难以动态调整语音（如实时调整语速、情感韵律或音乐渲染）；缺乏工具调用能力和上下文感知
<!-- This report presents Step-Audio, the first production-ready open-source framework
for intelligent speech interaction that harmonizes comprehension and generation
through four key innovations. -->
1. 提出 Step-Audio，通过四点创新统一理解和生成：
<!-- • 130B-Parameter Multi-modal Model: A single unified model integrating
comprehension and generation capabilities, performing speech recognition,
semantic understanding, dialogue, voice cloning, audio editing and speech
synthesis. We have made the 130B Step-Audio-Chat variant open source.
• Generative Data Engine: Eliminates traditional TTS’s reliance on manual
data collection by generating high-quality audio through our 130B-parameter
multi-modal model. Leverages this data to train and publicly release a
resource-efficient Step-Audio-TTS-3B model with enhanced instruction-
following capabilities for controllable speech synthesis.
• Granular Voice Control: Enables precise regulation through instruction-
based control design, supporting multiple emotions (anger, joy, sadness),
dialects (Cantonese, Sichuanese, etc.), and vocal styles (RAP/Singing, a
cappella humming) to meet diverse speech generation needs.
• Enhanced Intelligence: Improves agent performance in complex tasks
through ToolCall mechanism integration and role-playing enhancements. -->
    1. 130B 参数多模态模型，集成理解和生成能力，开源 Step-Audio-Chat 版本
    2. 生成式数据引擎，消除对手动数据采集的依赖，开源 Step-Audio-TTS-3B 模型
    3. 细粒度语音控制，支持多种情感、方言和声乐风格
    4. 增强智能，通过工具调用机制和角色扮演提升复杂任务处理能力
<!-- In open-source benchmarks, Step-Audio demonstrates exceptional performance. It
achieves SoTA results on open-domain question answering and complex instruction
tasks including LLaMA Question, TrivialQA, and ComplexBench, with an average
improvement of 9.3 points compared to the best open-source metrics, validating its
advantage in generalized deep semantic understanding capabilities. Additionally,
to address the current lack of comprehensive end-to-end speech dialogue evaluation
systems, we introduce the multi-dimensional StepEval-Audio-360 evaluation frame-
work covering 9 dimensions, including logical reasoning, creative ability, language
proficiency, and comprehension control among other key capabilities. As shown in Figure 1, Step-Audio achieves SoTA results across all dimensions in subjective
comparisons against open-source models like GLM-4-Voice and Qwen2-Audio,
with improvements of 19.2%, 23.7%, and 43.2% in response quality, response
relevance, and factual accuracy respectively. Particularly in generation control
dimensions such as emotion understanding, speech rate control, RAP vocals,
and role-playing, compared to open-source SoTA models, the IF (Instruction
Following) and MOS (Mean Opinion Score) metrics improved by 29.8% and 27.1%
respectively, highlighting its leading advantage in complex speech interaction
scenarios. -->
5. 在 LLaMA Question、TrivialQA 和 ComplexBench 等基准上，平均提升 9.3 个点；且超过 GLM-4-Voice 和 Qwen2-Audio 等模型

## 相关工作（略）

## 架构
<!-- Traditional voice dialogue systems typically employ a cascaded architecture com-
prising ASR, LLM, and TTS modules. However, our proposed model, having
undergone comprehensive multi-modal training and alignment of text and au-
dio during the pretraining phase, already possesses end-to-end voice dialogue
capabilities. Despite extensive exploration of alternative designs, we ultimately
adopted the AQTA (audio input, text output) + TTS framework for real-time
voice dialogue as shown in Figure 2, driven by the following considerations: -->
本文采用 AQTA（音频输入，文本输出）+ TTS 框架，如图：
![](image/Pasted%20image%2020250810152148.png)

采用此结构的原因如下：
<!-- Scarcity of high-quality pure-voice dialogue data: The limited avail-
ability of pure-voice dialogue data, coupled with its constrained scenarios,restricts the training efficiency of end-to-end voice dialogue models. -->
+ 缺乏高质量纯语音对话数据，限制语音对话模型的训练效率
<!-- Controllability and customization of output speech: By incorporating
a TTS module, we gain flexible control over speech parameters such as timbre
and pitch to meet users’ personalized demands, while continuously enhancing
the model’s expressive capabilities. -->
+ 输出语音的可控性和定制化：通过 TTS 模块，可以控制音色和音调等参数

<!-- Our goal is to establish Step-Audio as a real-time multi-modal model that seam-
lessly integrates speech understanding and synthesis through four key components:
(1) A dual-codebook tokenization framework employing parallel linguistic (16.7Hz,
1024-codebook) and semantic (25Hz, 4096-codebook) tokenizers with 2:3 temporal
interleaving; (2) A 130B-parameter LLM based on Step-1 (StepFun, 2024a), en-
hanced through audio-contextualized continual pretraining and postraining; (3) A
hybrid speech synthesizer combining with flow matching and neural vocoder, opti-
mized for real-time waveform generation. In addition, a Voice Activity Detection
(VAD) module was employed to extract vocal segments. -->
本文目标为建立 Step-Audio 实时多模态模型，通过以下模块实现语音理解和合成：
+ 双 codebook tokenization 框架，实现 linguistic（16.7Hz，1024-codebook）和 semantic（25Hz，4096-codebook）以 2:3 时间交错并行
+ 130B 参数 LLM，基于 Step-1 模型，采用音频上下文预训练和 postraining 
+ 混合语音合成，结合 flow matching 和 vocoder，优化实时波形生成
+ VAD 模块提取语音片段

### Tokenizer
<!-- To overcome the limitations of conventional speech tokenizers, which separately
capture information for understanding or generation task, we propose a dual-
codebook speech tokenizer framework in Step-Audio similar to ARCON (Ming
et al., 2024). This approach employs two distinct tokenizers, linguistic and
semantic, to better represent speech features. The linguistic tokenizer is utilized to
extract structured, high-level representations, including phonemic and linguistic
features, whereas the semantic tokenizer is designed to encode both semantic and
coarse-grained acoustic characteristics. -->
<!-- For linguistic tokenization, we utilize the output from the Paraformer (Z. Gao,
Zhang, McLoughlin, & Yan, 2022) encoder, which is quantized into discrete
representations at a token rate of 16.7 Hz. For semantic tokenization, we employ
CosyVoice’s (Du, Chen, et al., 2024) tokenizer, specifically designed to efficiently
encode features essential for generating natural and expressive speech outputs,
operating at a token rate of 25 Hz. The linguistic tokenizer employs a codebook
size of 1024, while the semantic tokenizer utilizes a larger codebook size of 4096
to capture finer acoustic details. -->
提出 双 codebook tokenization 框架：
+ linguistic tokenizer，提取结构化高层表征（如音素和语言特征），采用 Paraformer 编码器输出，量化为 16.7Hz 的离散表征，codebook 大小为 1024
+ semantic tokenizer，编码语义和粗粒度声学特征，采用 CosyVoice tokenizer，量化为 25Hz 的离散表征，codebook 大小为 4096
<!-- To effectively integrate these two tokenization schemes, we implement a token-
level interleaving approach inspired by SpiritLM (Nguyen et al., 2024). Given the
differing token rates, we establish a temporal alignment ratio of 2:3, where every
two linguistic tokens are paired with three semantic tokens. -->
采用了 token-level 的交错方法集成这两种 tokenization，比例为 2:3，即每两个 linguistic token 与三个 semantic token 配对。

### LLM
<!-- To enhance Step-Audio’s ability to effectively process speech information and
achieve accurate speech-text alignment, we conducted audio continual pretraining
based on Step-1, a 130-billion parameter pretrained text-based LLM. The details of
the pretrain and post-train processes for Step-Audio are comprehensively discussed
in section 4 and 5. -->
基于 Step-1 进行音频持续预训练（audio continual pretraining），以提高 Step-Audio 处理语音信息的能力和语音文本对齐的准确性。
<!-- In multi-turn dialogue systems, the substantial disparity in length between audio
tokens and text tokens necessitates efficient processing strategies. To address
this, historical information is initially transcribed into textual format utilizing an
ASR model prior to system input, thereby optimizing computational efficiency.
However, it should be noted that the model architecture maintains the capability
to process and utilize audio tokens as historical context when required. -->
多轮对话中音频 token 和文本 token 长度差异很大，因此历史信息在输入系统前先通过 ASR 转为文本。

### Decoder
<!-- Speech decoder consists of a 3-billion parameter language model, a flow-matching
model and a mel-to-wave vocoder primarily designed to receive text or audio
tokens and generate continuous time-domain stylized waveform that incorporate
historical information and instructions. To optimize the intelligibility and natu-
ralness of the synthesized speech, the speech decoder is trained using a dual-code
interleaving approach, ensuring seamless integration of linguistic and semantic
features throughout the generation process. On a speech decoder with a larger
parameter, we have observed the emergence of enhanced generative capabilities.
For further details, please refer to section 5.1. -->
Decoder 由 3B 参数的 LLM、flow-matching 模型和 mel-to-wave vocoder 组成，输入文本或音频 token，生成波形。采用 dual-code interleaving 方法训练 decoder，确保 linguistic 和 semantic 特征在生成过程中无缝集成。

### 实时推理
<!-- To enable real-time interactions, we have designed an optimized inference pipeline
as shown in Figure 3. At its core, the Controller module manages state transitions,
orchestrates speculative response generation, and ensures seamless coordination
between critical subsystems. These subsystems include VAD for detecting user
speech, the Streaming Audio Tokenizer for processing audio in real-time, the
Step-Audio language model and Speech Decoder for processing and generating
responses, and the Context Manager for preserving conversational continuity. -->
实时推理流程如图：
![](image/Pasted%20image%2020250810160419.png)

核心为 Controller 模块，管理状态转换、响应生成和子系统协调。子系统包括：
+ VAD：检测用户语音
+ Streaming Audio Tokenizer：实时处理音频
+ Step-Audio LLM 和 Speech Decoder：处理和生成响应
+ Context Manager：保持对话连续性

<!-- Speculative Response Generation To reduce interaction latency, the system
preemptively generates speculative responses. This minimizes perceived delays and
enhances responsiveness, though at the cost of occasional redundant computations
when speculative responses are discarded. The system begins in the Silence
state, awaiting user input. When the VAD detects active speech, the system
transitions to the UserSpeaking state. During this state, the Streaming Audio
Tokenizer begins converting audio into tokens. If the user momentarily pauses,
the system enters the UserPaused state, where speculative response generation
is triggered. By preemptively generating a response in anticipation of input
completion, the system reduces latency when the conversation resumes. If the
user resumes speaking, the speculative response is discarded. Once the system
confidently determines that the user has finished speaking, it transitions to the
BotReplying state, commits the most recent speculative response, and delivers
its audio output. If interrupted by user speech, the system prioritizes the new
input while maintaining conversational continuity. After completing its response,
the system returns to the Silence state, ready for the next interaction. Empirical
analysis shows that approximately 40% of speculative responses are successfully
committed. This mechanism reduces per-response latency by approximately 500ms
compared to non-speculative methods. -->
推理相应生成：系统预生成响应以减少延迟。从 Silence 状态开始，等待用户输入。当 VAD 检测到语音时，进入 UserSpeaking 状态，此时 Streaming Audio Tokenizer 开始将音频转换为 token。如果用户暂停，进入 UserPaused 状态，触发预生成响应。若用户继续说话，则丢弃预生成的响应。一旦确定用户说完，进入 BotReplying 状态，提交最新的预生成响应并输出音频。如果在此期间有新的用户输入，则优先处理新输入，同时保持对话连续性。完成响应后返回 Silence 状态，准备下一次交互。
<!-- Context Management Our system utilizes text transcription instead of raw
audio tokens for historical context, as it provides a more compact representation
(with an average text-to-audio token ratio of 1:14), improving performance, and en-
abling longer conversations with minimal impact on quality. ASR asynchronously
transcribes user speech into text, maintaining an accurate and up-to-date conver-
sation history. -->
上下文管理：系统使用文本作为历史上下文（提高性能并支持更长对话）。ASR 异步将用户语音转为文本，得到对话历史。
<!-- Streaming Audio Tokenizer The input audio stream is processed through
two parallel tokenizer pipelines, each employing fixed-duration segmentation.
The resulting tokens are seamlessly merged into a single sequence with a 2:3
interleaving ratio. Without the streaming audio tokenizer, the inference time will
be significantly slower, depending on the length of the audio input. -->
Streaming Audio Tokenizer：输入音频通过两个并行的 tokenizer，两类 token 以 2:3 的比例合并为一个序列。

## 预训练

### 数据集
<!-- Our multi-modal pretraining dataset integrates three major categories of data
resources: audio, text, and images. The audio section comprises 1.1 trillion
tokens of audio continuation data (approximately 7,300,000 hours), 113 billion
tokens of TTS (Text-to-Speech) synthesized speech data (about 700,000 hours),
105 billion tokens of ASR (Automatic Speech Recognition) data (around 650,000
hours), and 350 billion tokens of audio-text alternating data (approximately
2,000,000 hours). The text data, amounting to 800 billion tokens, encompasses
web documents, books, code, and proprietary materials. The image section
includes 800 billion tokens of image-text paired/alternating data, sourced from
web pages, books, and proprietary resources. -->
多模态预训练数据集包含三类数据：
+ 音频：1.1 万亿 token 的音频续写数据（约 730 万小时）、1130 亿 token 的 TTS 合成语音数据（约 70 万小时）、1050 亿 token 的 ASR 数据（约 65 万小时）、3500 亿 token 的音频文本交替数据（约 200 万小时）
+ 文本：8000 亿 token，包含网页、书籍、代码和专有材料
+ 图像：8000 亿 token 的图像文本配对/交替数据，来源于网页、书籍和专有资源

### 训练细节
<!-- Step-Audio is a component of Step-Omni, which is designed to train a unified
pretrained model for speech, image, and text. This training is based on a pretrained
text model and image encoder for continued pretraining. The entire process is
divided into three stages in total.
• Stage1: We expanded the vocabulary of the pretrained text model by
adding 5,120 audio tokens and integrated a pretrained image encoder to
form the Step-Omni model. During training, to ensure minimal loss of the
text model’s capabilities, the learning rate of the text model backbone is
maintained at a low level (2e-5) throughout. However, the learning rates for
the embedding and language model (LM) head are set five times higher than
the backbone’s to facilitate faster convergence of the newly added tokens.
Meanwhile, the image encoder remains frozen during the entire training
process. At this stage, audio, text, and image data are used in a 2:1:1 ratio,
with audio data consisting solely of pure audio continuation tasks.
• Stage2: After training on 1.2T tokens in the stage1 phase, we incorporate
audio-text interleaved data for further training, with a 1:1 ratio of audio
continuation data to audio-text interleaved data. During this stage, the
ratio of audio, text, and image data remains 2:1:1.
• Stage3: After training on 800B tokens in the stage2 phase, we incorporate
ASR and TTS data for further training. The ratio of audio continuation
data, audio-text interleaved data, ASR data, and TTS data is set to 1:1:1:1.
During this phase, the ratio of audio, text, and image data is adjusted to
4:3:3. Additionally, the learning rates for the embedding and LM head are
synchronized with the backbone, utilizing a cosine schedule that decreases
from 2e-5 to 5e-6.
We employ the same pre-training strategy across models of varying parameter
scales. -->
Step-Audio 是 Step-Omni 的一部分，分为三个阶段训练：
+ 阶段一：扩展预训练文本模型的词表，添加 5120 个音频 token，并集成预训练图像编码器，形成 Step-Omni 模型。训练时，文本模型的学习率保持在 2e-5，嵌入和语言模型头的学习率是其五倍。此阶段使用音频、文本和图像数据的比例为 2:1:1，仅使用纯音频续写任务。
+ 阶段二：在阶段一训练 1.2T token 后，加入音频文本交替数据，音频续写数据和音频文本交替数据的比例为 1:1。此阶段音频、文本和图像数据的比例仍为 2:1:1。
+ 阶段三：在阶段二训练 800B token 后，加入 ASR 和 TTS 数据，音频续写数据、音频文本交替数据、ASR 数据和 TTS 数据的比例为 1:1:1:1。此阶段音频、文本和图像数据的比例调整为 4:3:3。嵌入和语言模型头的学习率与主干同步，使用从 2e-5 到 5e-6 的余弦调度。
+ 在不同参数规模的模型上采用相同的预训练策略。

## 后训练

### TTS

### AQTA

## 评估（略）
