> 


文本前端的作用是，将输入的原始文本转换为音素序列，通常包含几个步骤：
+ 文本正则化：

对于英文，以  [tacotron](https://github.com/keithito/tacotron/tree/master) 中的 text 模块为例，其中包含四个文件：
+ [cleaners.py](https://github.com/keithito/tacotron/blob/master/text/cleaners.py "cleaners.py")
+ [cmudict.py](https://github.com/keithito/tacotron/blob/master/text/cmudict.py "cmudict.py")
+ [numbers.py](https://github.com/keithito/tacotron/blob/master/text/numbers.py "numbers.py")
+ [symbols.py](https://github.com/keithito/tacotron/blob/master/text/symbols.py "symbols.py")
以及一个 [__init__.py](https://github.com/keithito/tacotron/blob/master/text/__init__.py "__init__.py") 文件。

下面以一段文本为例，详细描述此过程。

首先 cd 到 tacotron 所在的目录下，在 `__init__.py` 文件中添加测试代码：
```python

if __name__ == "__main__":

text = "Hello world!"

sequence = text_to_sequence(text, ["english_cleaners"])

print(sequence)
```
然后执行代码：
```bash
python -m text.__init__
```
打印结果为：
```
[35, 32, 39, 39, 42, 64, 50, 42, 45, 39, 31, 54, 1]
```

下面分析此过程。对于给定的文本 "Hello world!"，在不考虑文本包含花括号括起来的 ARPAbet 序列时，执行 `text_to_sequence` 用于将给定的文本转为 phoneme 序列：
+ 此时先执行 `_clean_text` 函数，根据给定的 `cleaner_names` 对文本进行正则化 `text = cleaner(text)` 用于从 [cleaners.py](https://github.com/keithito/tacotron/blob/master/text/cleaners.py "cleaners.py") 中查找对应语种的 cleaner，对于英文，其执行 `english_cleaners` 函数
+ `english_cleaners` 函数包含五个部分，分别是：
	+ `convert_to_ascii`：将文本转为 unidecode 编码格式，避免
	+ `lowercase`
	+ `expand_numbers`
	+ `expand_abbreviations`
	+ `collapse_whitespace`


