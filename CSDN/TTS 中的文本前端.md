> 本文主要介绍经典 TTS 模型中的文本前端，以 tacotron、Glow-TTS 和 VITS 为例，详细描述了文本前端的处理过程。

介绍之前首先需要了解两个概念，phoneme（音素）和 grapheme（字素）：
+ phoneme（音素）：指最小的发音单元，为语音的最小单位
+ grapheme（字素）：为书面语言的最小书写单位

还需要了解：
+ ARPAbet 字符集：[ARPABET](https://en.wikipedia.org/wiki/ARPABET)（也拼写为ARPAbet）是美国国防高级研究计划局（ARPA，其前身）作为语音理解项目（1971-1976 年）的一部分开发的语音字母。将通用美式英语中的音素和音位变体表示为不同的ASCII字符
+ CMUDict：[CMUDict](http://www.speech.cs.cmu.edu/cgi-bin/cmudict)（卡内基梅隆大学发音词典）是一个用于英语单词的发音字典，包含了大量英语词汇及其标准发音。它由卡内基梅隆大学的语言技术研究团队创建，并被广泛应用于语音识别、语音合成、自然语言处理等领域；使用的音素集基于 Arpabet 音标系统，其标准版本包含 39 个音素

一般来说，文本前端的作用是，将输入的原始文本转换为音素（字素）序列，通常包含几个步骤：
+ 文本正则化
+ text-to-phoneme（grapheme） 转换

## Tacotron 中的文本前端

对于英文，以  [tacotron](https://github.com/keithito/tacotron/tree/master) 中的 text 模块为例，其中包含四个文件：
+ [cleaners.py](https://github.com/keithito/tacotron/blob/master/text/cleaners.py "cleaners.py")
+ [cmudict.py](https://github.com/keithito/tacotron/blob/master/text/cmudict.py "cmudict.py")
+ [numbers.py](https://github.com/keithito/tacotron/blob/master/text/numbers.py "numbers.py")
+ [symbols.py](https://github.com/keithito/tacotron/blob/master/text/symbols.py "symbols.py")
以及一个 [__init__.py](https://github.com/keithito/tacotron/blob/master/text/__init__.py "__init__.py") 文件。

下面以一段文本为例，详细描述此过程。

首先 cd 到 tacotron 所在的目录下，在 [__init__.py](https://github.com/keithito/tacotron/blob/master/text/__init__.py "__init__.py") 文件中添加测试代码：
```python
if __name__ == "__main__":
  text = "Hello world!"
  sequence = text_to_sequence(text, ["english_cleaners"])
  print(sequence)
```
然后执行代码：
```bash
cd /path/to/tacotron
python -m text.__init__
```
打印结果为：
```
[35, 32, 39, 39, 42, 64, 50, 42, 45, 39, 31, 54, 1]
```

下面分析此过程。

考虑一段复杂文本 "**This is a test with non-ASCII characters like café, UPPERCASE letters, 123 numbers, abbreviations like Dr. and Mr., and   extra   spaces.**"，在不考虑文本包含花括号括起来的 ARPAbet 序列时，执行 `text_to_sequence` 用于将给定的文本转为 phoneme 序列，其核心代码如下：
```python
from text import cleaners
from text.symbols import symbols

_symbol_to_id = {s: i for i, s in enumerate(symbols)}

def _clean_text(text, cleaner_names):
  for name in cleaner_names:
    cleaner = getattr(cleaners, name)
    if not cleaner:
      raise Exception('Unknown cleaner: %s' % name)
    text = cleaner(text)
  return text

def _symbols_to_sequence(symbols):
  return [_symbol_to_id[s] for s in symbols if _should_keep_symbol(s)]

def _should_keep_symbol(s):
  return s in _symbol_to_id and s is not '_' and s is not '~'

def text_to_sequence(text, cleaner_names):
  sequence = []
  equence += _symbols_to_sequence(_clean_text(text, cleaner_names))
  sequence.append(_symbol_to_id['~'])
  return sequence
```
执行测试代码，流程如下：
+ 先运行 `_clean_text` 函数，根据给定的 `cleaner_names` 对文本进行正则化 `text = cleaner(text)` 用于从 [cleaners.py](https://github.com/keithito/tacotron/blob/master/text/cleaners.py "cleaners.py") 中查找对应语种的 cleaner，对于英文，其执行 `english_cleaners` 函数
+ `english_cleaners` 函数包含五个部分，分别是：
	+ `convert_to_ascii`：将文本转为 unidecode 编码格式，避免出现特殊字符（主要将一些包含非ASCII字符，如重音符号、中文、日文等，的 Unicode 文本转换为 ASCII 近似字符，例如，对于中文，将“中国”转为“Zhong Guo”，对于日文，将“わかりました”转为“wakarimashita”）
	+ `lowercase`：将文本转为小写
	+ `expand_numbers`：实际执行的是 [numbers.py](https://github.com/keithito/tacotron/blob/master/text/numbers.py "numbers.py") 中的 `normalize_numbers` 函数，采用正则表达式将文本中的数字转为英文单词，例如，将数字 "123" 转为 "one hundred twenty-three"，将数字 "12.34" 转为 "twelve point thirty-four"
	+ `expand_abbreviations`：将文本中的缩写转为全称，其中的缩写和全称对应关系在 [cleaners.py](https://github.com/keithito/tacotron/blob/master/text/cleaners.py "cleaners.py") 中的 `_abbreviations` 字典中定义，例如，将 "dr. Zhang is a doctor, mr. Smith is a mister, and mrs. Johnson is a misess." 转为 "doctor Zhang is a doctor, mister Smith is a mister, and misess Johnson is a misess."（注意缩写需要包含句点 "."）
	+ `collapse_whitespace`：将文本中的多个空格转为一个空格

完成上述的文本正则化后，得到了纯 unicode 编码下的可以直接转为 phoneme 序列的文本。对于上述示例，得到的结果为："**this is a test with non-ascii characters like cafe, uppercase letters, one hundred twenty-three numbers, abbreviations like doctor and mister, and extra spaces.**"
> 可以看到，文本中的 "café" 被转为 "cafe"，"Dr." 被转为 "doctor"，"Mr." 被转为 "mister"，"Mrs." 被转为 "misess"，"123" 被转为 "one hundred twenty-three"，多余的空格被转为一个空格。

接下来执行 [__init__.py](https://github.com/keithito/tacotron/blob/master/text/__init__.py "__init__.py") 文件中的 `_symbols_to_sequence` 函数，将文本转为 phoneme 序列，这部分核心代码如下：
```python
from text.symbols import symbols
_symbol_to_id = {s: i for i, s in enumerate(symbols)}
def _symbols_to_sequence(symbols):
    return [_symbol_to_id[s] for s in symbols if _should_keep_symbol(s)]
def _should_keep_symbol(s):
    return s in _symbol_to_id and s is not "_" and s is not "~"
```
+ 首先判断得到的字符串是否为特殊字符 `_` 和 `~`，如果是，则直接跳过
+ 对于其他的字符，查找位于 [symbols.py](https://github.com/keithito/tacotron/blob/master/text/symbols.py "symbols.py") 中定义的 `symbols` 列表中的索引，将其转为对应的 id。

> tacotron 的 symbols 定义如下（总长度为 149），其中 phoneme 序列的个数为 84：
```python
symbols = ['_', '~', 'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z', 'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', '!', "'", '(', ')', ',', '-', '.', ':', ';', '?', ' ', '@AA', '@AA0', '@AA1', '@AA2', '@AE', '@AE0', '@AE1', '@AE2', '@AH', '@AH0', '@AH1', '@AH2', '@AO', '@AO0', '@AO1', '@AO2', '@AW', '@AW0', '@AW1', '@AW2', '@AY', '@AY0', '@AY1', '@AY2', '@B', '@CH', '@D', '@DH', '@EH', '@EH0', '@EH1', '@EH2', '@ER', '@ER0', '@ER1', '@ER2', '@EY', '@EY0', '@EY1', '@EY2', '@F', '@G', '@HH', '@IH', '@IH0', '@IH1', '@IH2', '@IY', '@IY0', '@IY1', '@IY2', '@JH', '@K', '@L', '@M', '@N', '@NG', '@OW', '@OW0', '@OW1', '@OW2', '@OY', '@OY0', '@OY1', '@OY2', '@P', '@R', '@S', '@SH', '@T', '@TH', '@UH', '@UH0', '@UH1', '@UH2', '@UW', '@UW0', '@UW1', '@UW2', '@V', '@W', '@Y', '@Z', '@ZH']
```

到这里就完成了 tacotron 中的文本前端处理过程，得到了可以供给模型的输入序列。

## Grow-TTS、Grad-TTS 中的文本前端

Tacotron 并没有提取文本的音素序列，导致提取的特征不包含发音等信息。其 text 模块中的 [cmudict.py](https://github.com/keithito/tacotron/blob/master/text/cmudict.py "cmudict.py") 也仅是用于手动给出一些单词的 phoneme，从而影响最终合成的音频效果。

下面考虑 [Glow-TTS](https://github.com/jaywalnut310/glow-tts/tree/master) 中的文本前端处理过程（[Grad-TTS](https://github.com/huawei-noah/Speech-Backbones/tree/main/Grad-TTS) 同理），相比于 tacotron，Glow-TTS 采用了 grapheme-to-phoneme (G2P) 模型，用于将文本转为音素序列。

同理，在 text 模块中的 [__init__.py](https://github.com/jaywalnut310/glow-tts/blob/master/text/__init__.py "__init__.py") 中给出如下的测试代码：
```python
if __name__ == "__main__":
    from text import cmudict

    text = "hello world"
    sequence = text_to_sequence(
        text, ["english_cleaners"], cmudict.CMUDict("./data/cmu_dictionary")
    )
    print(len(symbols))
    print(sequence)
```

然后执行代码：
```bash
cd /path/to/glow-tts
python -m text.__init__
```
打印结果为：
```
[106, 73, 117, 123, 11, 144, 98, 117, 90]
```

其 `text_to_sequence` 函数核心部分如下：
```python
from text import cleaners
from text.symbols import symbols

_symbol_to_id = {s: i for i, s in enumerate(symbols)}

def get_arpabet(word, dictionary):
  word_arpabet = dictionary.lookup(word)
  if word_arpabet is not None:
    return "{" + word_arpabet[0] + "}"
  else:
    return word

def text_to_sequence(text, cleaner_names, dictionary=None):
  sequence = []
  space = _symbols_to_sequence(' ')
  # Check for curly braces and treat their contents as ARPAbet:
  clean_text = _clean_text(text, cleaner_names)
  if dictionary is not None:
    clean_text = [get_arpabet(w, dictionary) for w in clean_text.split(" ")]
    for i in range(len(clean_text)):
      t = clean_text[i]
      if t.startswith("{"):
      sequence += _arpabet_to_sequence(t[1:-1])
    else:
      sequence += _symbols_to_sequence(t)
      sequence += space
  else:
      sequence += _symbols_to_sequence(clean_text)

  # remove trailing space
  if dictionary is not None:
    sequence = sequence[:-1] if sequence[-1] == space[0] else sequence
  return sequence
```

其中的 `get_arpabet` 函数用于将单词根据给定的字典转为 ARPAbet 序列；`text_to_sequence` 函数中的 `dictionary` 参数用于给出单词的 ARPAbet 序列，如果不给出，则只进行文本的正则化，其最终的效果和 tacotron 中的 `text_to_sequence` 函数类似。具体来说，`text_to_sequence` 函数的流程如下：
+ 先执行 `_clean_text` 函数，对文本进行正则化，参考 tacotron 中的 `english_cleaners` 函数
+ 如果给出了 `dictionary` 参数，则对正则化后的每个单词调用 `get_arpabet` 函数：
    + 参数 `dictionary` 本质为一个字典，可以从 [CMUDict 官网](http://www.speech.cs.cmu.edu/cgi-bin/cmudict) 下载，其中包含了大量的单词和其对应的 ARPAbet 音素序列，例如，对于单词 "hello"，其对应的 ARPAbet 序列为 "HH AH0 L OW1"，可以通过 `dictionary.lookup("hello")` 得到
    + 如果单词位于字典中，则返回 `{ARPAbet}` 形式的字符串，代表了此单词的发音
    + 否则，返回原单词
+ 最终得到的格式形如 `["{ARPAbet1}", "{ARPAbet2}", ...]`，例如，对于文本 "hello world"，得到的结果为 `["{HH AH0 L OW1}", "{W ER1 L D}"]`
+ 对于得到的 ARPAbet 序列，执行 `_arpabet_to_sequence` 函数，将其转为音素序列
    + `_arpabet_to_sequence` 对所有的 ARPAbet 序列前面加上 `@`，例如，对于 ARPAbet 序列 "HH AH0 L OW1"，其转为音素序列为 `["@HH", "@AH", "@L", "@OW"]`
    + 然后和之前一样，执行 `_symbols_to_sequence` 函数，将其转为音素序列，得到对应的 id。

> Glow-TTS 中的 symbols 定义如下（总长度为 148），其中 phoneme 序列的个数为 84：
```python
symbols = ['_', '-', '!', "'", '(', ')', ',', '.', ':', ';', '?', ' ', 'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z', 'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', '@AA', '@AA0', '@AA1', '@AA2', '@AE', '@AE0', '@AE1', '@AE2', '@AH', '@AH0', '@AH1', '@AH2', '@AO', '@AO0', '@AO1', '@AO2', '@AW', '@AW0', '@AW1', '@AW2', '@AY', '@AY0', '@AY1', '@AY2', '@B', '@CH', '@D', '@DH', '@EH', '@EH0', '@EH1', '@EH2', '@ER', '@ER0', '@ER1', '@ER2', '@EY', '@EY0', '@EY1', '@EY2', '@F', '@G', '@HH', '@IH', '@IH0', '@IH1', '@IH2', '@IY', '@IY0', '@IY1', '@IY2', '@JH', '@K', '@L', '@M', '@N', '@NG', '@OW', '@OW0', '@OW1', '@OW2', '@OY', '@OY0', '@OY1', '@OY2', '@P', '@R', '@S', '@SH', '@T', '@TH', '@UH', '@UH0', '@UH1', '@UH2', '@UW', '@UW0', '@UW1', '@UW2', '@V', '@W', '@Y', '@Z', '@ZH']
```

## VITS 中的文本前端

对于 vits 中的文本前端，其中的文本正则化和前面的两个模型大差不差。
同理在 [__init__.py](https://github.com/jaywalnut310/vits/blob/main/text/__init__.py "__init__.py") 中给出下面的测试代码：
```python
if __name__ == "__main__":
    text = "hello world"
    sequence = text_to_sequence(text, ["english_cleaners2"])
    print(sequence)
```

然后执行代码：
```bash
cd /path/to/vits
python -m text.__init__
```

打印结果为：
```
[50, 83, 54, 156, 57, 135, 16, 65, 156, 87, 158, 54, 46]
```

其 text 模块中的 [__init__.py](https://github.com/jaywalnut310/vits/blob/main/text/__init__.py "__init__.py") 文件在 `text_to_sequence` 函数的核心代码如下：
```python
from text import cleaners
from text.symbols import symbols

_symbol_to_id = {s: i for i, s in enumerate(symbols)}

def text_to_sequence(text, cleaner_names):

  sequence = []
  clean_text = _clean_text(text, cleaner_names)
  for symbol in clean_text:
    symbol_id = _symbol_to_id[symbol]
    sequence += [symbol_id]
  return sequence

def _clean_text(text, cleaner_names):
  for name in cleaner_names:
    cleaner = getattr(cleaners, name)
    if not cleaner:
      raise Exception('Unknown cleaner: %s' % name)
    text = cleaner(text)
  return text
```

其直接调用 `_clean_text` 函数，这部分的代码位于 [cleaners.py](https://github.com/jaywalnut310/vits/blob/main/text/cleaners.py "cleaners.py") 中，对于 `english_cleaners`，其核心如下：
```python
from phonemizer import phonemize

def english_cleaners(text):
  '''Pipeline for English text, including abbreviation expansion.'''
  text = convert_to_ascii(text)
  text = lowercase(text)
  text = expand_abbreviations(text)
  phonemes = phonemize(text, language='en-us', backend='espeak', strip=True)
  phonemes = collapse_whitespace(phonemes)
  return phonemes


def english_cleaners2(text):
  '''Pipeline for English text, including abbreviation expansion. + punctuation + stress'''
  text = convert_to_ascii(text)
  text = lowercase(text)
  text = expand_abbreviations(text)
  phonemes = phonemize(text, language='en-us', backend='espeak', strip=True, preserve_punctuation=True, with_stress=True)
  phonemes = collapse_whitespace(phonemes)
  return phonemes
```

其中的 `convert_to_ascii`、`lowercase`、`expand_abbreviations` 和 `collapse_whitespace` 函数和 tacotron 中的类似，不再赘述。不同的是，vits 中的 `english_cleaners` 函数调用了 `phonemize` 函数，用于将文本转为音素序列。这里的 `phonemize` 来自第三方库 [phonemizer](https://github.com/bootphon/phonemizer)，主要用于将给定的文本转为音素序列。

> phonemizer 的使用方法如下：
```python
from phonemizer import phonemize

text = "hello world"
phonemized_text = phonemize(text, backend='espeak', language='en-us', strip=True, preserve_punctuation=True, with_stress=True)
print(phonemized_text)
# 输出为：
həlˈoʊ wˈɜːld
```

这里得到的是 IPA 格式的音素序列，需要进一步转为对应的 id，具体的转换过程和前面的两个模型类似，不再赘述。

> VITS 中的 symbols 定义如下（总长度为 178），其中包含
+  1 个 _pad 符号，用于填充
+ 16 个 _punctuation 符号，用于标点符号
+ 52 个 _letters 符号，为 26*2 个字母（大小写）
+ 109 个 _letters_ipa 符号，为 IPA 格式的音素集
```python
symbols = ['_', ';', ':', ',', '.', '!', '?', '¡', '¿', '—', '…', '"', '«', '»', '“', '”', ' ', 'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z', 'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', 'ɑ', 'ɐ', 'ɒ', 'æ', 'ɓ', 'ʙ', 'β', 'ɔ', 'ɕ', 'ç', 'ɗ', 'ɖ', 'ð', 'ʤ', 'ə', 'ɘ', 'ɚ', 'ɛ', 'ɜ', 'ɝ', 'ɞ', 'ɟ', 'ʄ', 'ɡ', 'ɠ', 'ɢ', 'ʛ', 'ɦ', 'ɧ', 'ħ', 'ɥ', 'ʜ', 'ɨ', 'ɪ', 'ʝ', 'ɭ', 'ɬ', 'ɫ', 'ɮ', 'ʟ', 'ɱ', 'ɯ', 'ɰ', 'ŋ', 'ɳ', 'ɲ', 'ɴ', 'ø', 'ɵ', 'ɸ', 'θ', 'œ', 'ɶ', 'ʘ', 'ɹ', 'ɺ', 'ɾ', 'ɻ', 'ʀ', 'ʁ', 'ɽ', 'ʂ', 'ʃ', 'ʈ', 'ʧ', 'ʉ', 'ʊ', 'ʋ', 'ⱱ', 'ʌ', 'ɣ', 'ɤ', 'ʍ', 'χ', 'ʎ', 'ʏ', 'ʑ', 'ʐ', 'ʒ', 'ʔ', 'ʡ', 'ʕ', 'ʢ', 'ǀ', 'ǁ', 'ǂ', 'ǃ', 'ˈ', 'ˌ', 'ː', 'ˑ', 'ʼ', 'ʴ', 'ʰ', 'ʱ', 'ʲ', 'ʷ', 'ˠ', 'ˤ', '˞', '↓', '↑', '→', '↗', '↘', "'", '̩', "'", 'ᵻ']
```

## 常见的文本前端和音素提取库

常用的文本前端库：
1. [nltk](https://github.com/nltk/nltk)
2. 

常用的音素提取库：
1. [phonemizer](https://github.com/bootphon/phonemizer) 是一个用于将文本转换为音素表示的 Python 库，支持多种语言和多个音标系统。
2. [espeak-ng](https://github.com/espeak-ng/espeak-ng) 是一个基于开源文本转语音引擎 espeak 的 Python 包，支持多种语言的发音转换。支持 ARPAbet 和 IPA 等音标系统。
3. [python-pinyin](https://github.com/mozillazg/python-pinyin) 用于将汉字转为拼音。可以用于汉字注音、排序、检索。
4. [Merlin](https://github.com/CSTR-Edinburgh/merlin) 提供了 TTS 核心的声学建模模块（声学和语音特征归一化，神经网络声学模型训练和生成）。
