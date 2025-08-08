> Preprint 2025，bilibili
<!-- 翻译&理解 -->
<!-- Large-scale text-to-speech (TTS) models are typically categorized into autoregres-
sive and non-autoregressive systems. Although autoregressive systems exhibit
certain advantages in speech naturalness, their token-by-token generation mecha-
nism makes it difficult to precisely control the duration of the synthesized speech.
This becomes a significant limitation in applications such as video dubbing, where
strict audio-visual synchronization is required. This paper introduces IndexTTS2,
which proposes a novel, general, and autoregressive-model-friendly method for
speech duration control. The method supports two generation modes: one allows
explicit specification of the number of generated tokens, thereby enabling precise
control over speech duration; the other does not require manual token count in-
put, letting the model freely generate speech in an autoregressive manner while
faithfully reproducing prosodic characteristics from the input prompt. Furthermore,
IndexTTS2 achieves disentanglement between emotional expression and speaker
identity, enabling independent control of timbre and emotion. In the zero-shot
setting, the model is capable of perfectly reproducing the emotional characteristics
inherent in the input prompt. Additionally, users may provide a separate emotion
prompt (which can originate from a different speaker than the timbre prompt),
thereby enabling the model to accurately reconstruct the target timbre while con-
veying the specified emotional tone. In order to enhance the clarity of speech
during strong emotional expressions, we incorporate GPT latent representations
to improve the stability of the generated speech. Meanwhile, to lower the barrier
for emotion control, we design a soft instruction mechanism based on textual
descriptions by fine-tuning Qwen3. This facilitates the effective guidance of speech
generation with the desired emotional tendencies through natural language input.
Finally, experimental results on multiple datasets demonstrate that IndexTTS2
outperforms existing state-of-the-art zero-shot TTS models in terms of word error
rate, speaker similarity, and emotional fidelity. To promote further research and
facilitate practical adoption, we will release both the model weights and inference
code, enabling the community to reproduce and build upon our work. -->
