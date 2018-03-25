---
title: 文本聚类系列教程：（二）jieba中文分词工具进阶
author: Leno
tags: 文本聚类
mathjax: true
category: 文本聚类
thumbnail: 'https://i.loli.net/2018/03/18/5aae7476a5f07.png'
abbrlink: 931939a5
date: 2018-03-19 19:57:19
---

>jieba中文分词工具使用进阶篇，废话不多说吗，我们开始本次的学习吧~

----------

# 如何让分词的更加准确 #

我们之前举得例子有些文本其实很简单，我们后来确实换了官方的测试文本《围城》，但是均没避免一个问题，这些测试例都十分地中规中矩。在实际中需要我们做分词的文本可能是多种多样的，这时候的切词有可能会不太特别理想，导致分词的不准确。

那我们不妨下一个别的电子书（这里我下载的是《斗破苍穹》，为了测试我只用了第一章的文本），然后再进行切词，看下是否存在这样的问题。这里我们稍微改改上次的去停用词的代码，代码如下：

```python
import sys
import jieba
from os import path

d = path.dirname(__file__) # 获取当前文件的dir路径

text_path = 'txt/chapter2.txt' #《斗破苍穹》第一章的文本路径
text = open(path.join(d, text_path),'rb').read()

def CutWords(text):
    mywordlist = []
    seg_list = jieba.cut(text, cut_all=False)
    liststr="/ ".join(seg_list) # 添加切分符
    for myword in liststr.split('/'):
        if len(myword.strip())>1:
            mywordlist.append(myword)
    return ''.join(mywordlist) #返回一个字符串

txt5 = CutWords(text)
text_write = 'txt/5.txt'
with open(text_write,'w') as f:
    f.write(txt5)
    print("Success")

```

**结果如下：**

![result_cutwords.png](https://i.loli.net/2018/03/20/5ab12557a30b9.png)

终于被我们找到了一个切词错误，原文是这样的：

萧媚脑中忽然浮现出三年前那意气风发的少年

按照我们正常的断句，应为：

萧媚/脑中/忽然/浮现....，而jieba却认为“萧媚脑”是一个单词，从而导致此处分词不理想。

jieba考虑了这种情况，而且有很多的应对方案，下面我们先说最简单的。

# 调整词典 #

## 方法1：动态修改词典 ##

使用add_word(word,freq=None,tag=None)和del_word(word)可在程序中动态的修改词典，具体操作如下：

```python
import sys
import jieba
from os import path

d = path.dirname(__file__) # 获取当前文件的dir路径

# 此处增加代码
jieba.add_word('脑中')

  ····

```

**结果如下：**

![add_word_test.png](https://i.loli.net/2018/03/22/5ab3a2f27d481.png)

果然，这样的方法很直接的把我们原来切错的词变成了正确的词。与add_word()相对应的是delete_word()方法，根据字面意思我们也很容易理解delete_word()方法的作用，这里我就不做过多的演示了，大家在实际场景中直接运用就好了。

## 方法2：调节词频 ##

使用suggest_freq(segment, tune=True)调节单个词语的词频，使得它更容易被分出来，或者不被分出来。

但是需要注意的是：**自动计算的词频在使用 HMM 新词发现功能时可能无效。**

所以此时我们在做切词的时候需要把是HMM置为False。我们看下官方给的Demo（如果关闭HMM，很多新发现的词都消失了，所以‘萧媚脑’也消失了，无法做测试，我们的例子也是为了方便大家理解，所以也没必要非得针对这一个词做词频调节），具体的做法如下：

```python
import jieba

print('/'.join(jieba.cut('如果放到post中将出错。', HMM=False)))

jieba.suggest_freq(('中', '将'), True)

print('/'.join(jieba.cut('如果放到post中将出错。', HMM=False)))

print('/'.join(jieba.cut('「台中」正确应该不会被切开', HMM=False)))

jieba.suggest_freq('台中', True)

print('/'.join(jieba.cut('「台中」正确应该不会被切开', HMM=False)))
```

**结果：**

![suggest_freq.png](https://i.loli.net/2018/03/22/5ab3b191d1bfd.png)

对比下结果，不难发现suggest_freq()的使用方法，通过这样的强调高频词和低频词的方法可以做到分词更准确。

# 添加自定义词典 #

比起默认的词典，我们自定义的词典更适合我们自己的文本，这一点是毋庸置疑的。

词典格式和 dict.txt 一样，一个词占一行；每一行分三部分：词语、词频（可省略）、词性（可省略），用空格隔开，顺序不可颠倒。file_name 若为路径或二进制方式打开的文件，则文件必须为 UTF-8 编码。

这里我们的词典为：

```
云计算 5
李小福 2 nr
创新办 3 i
easy_install 3 eng
好用 300
韩玉赏鉴 3 nz
八一双鹿 3 nz
台中
凱特琳 nz
Edu Trust认证 2000
```

我们这个例子也用官方的Demo，代码如下：

```python
import sys
sys.path.append("../")
import jieba
jieba.load_userdict("userdict.txt")
# jieba在0.28版本之后采用延迟加载方式
# “import jieba”不会立即触发词典的加载，而是在有必要的时候才会加载词典
# 如果想手动加载，可执行代码： jieba.initialize() 进行手动初始化操作
# 也正是有了延迟加载机制，我们现在可以改变主词典的路径：
# jieba.set_dictionary('data/dict.txt.big')
# 官方还提供了占用内存较小的词典和适用于繁体字的词典，均在官方的GitHub上，有需要的可以自行下载。

import jieba.posseg as pseg
# pseg切分可以显示词性

# 以下三个操作是修改词典的巩固
jieba.add_word('石墨烯')
jieba.add_word('凱特琳')
jieba.del_word('自定义词')


test_sent = (
"李小福是创新办主任也是云计算方面的专家; 什么是八一双鹿\n"
"例如我输入一个带“韩玉赏鉴”的标题，在自定义词库中也增加了此词为N类\n"
"「台中」正確應該不會被切開。mac上可分出「石墨烯」；此時又可以分出來凱特琳了。"
)
words = jieba.cut(test_sent)
print('/'.join(words))

print("="*40)

result = pseg.cut(test_sent)

for w in result:
    print(w.word, "/", w.flag, ", ", end=' ')
```

**结果如下：**

![userdict.png](https://i.loli.net/2018/03/23/5ab46d77c08dc.png)

像‘云计算’、‘创新办’等词在没加载词典的时候是不能被识别出来的。像‘石墨烯’等在没有add_word()的时候也是不能识别出来的。可见效果还是不错的。

# 并行分词 #

原理：将目标文本按行分隔后，把各行文本分配到多个 Python 进程并行分词，然后归并结果，从而获得分词速度的可观提升

但是令人遗憾的是，这个模块并不支持Windows平台，原因是因为jieba的该模块是基于python自带的 multiprocessing 模块，而这个模块并不支持Windows。这里我就贴一下用法，使用Linux系统的同学可以自行体验下这个可观的速度提升。

**用法：**

* jieba.enable_parallel(4) # 开启并行分词模式，参数为并行进程数
* jieba.disable_parallel() # 关闭并行分词模式

# 最后 #

以上所讲的内容在日常的使用中应该是够用了，当然像基于TextRank算法的关键词抽取等内容，我这里并没涉及，并不是因为不重要，而是我对这个算法还不是很了解，硬着头皮写肯定也是照本宣科，效果肯定很差，所以先挖个坑吧，以后再填。

感谢阅读~