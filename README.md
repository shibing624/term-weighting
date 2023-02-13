# pke_zh
[![PyPI version](https://badge.fury.io/py/pke_zh.svg)](https://badge.fury.io/py/pke_zh)
[![Downloads](https://pepy.tech/badge/pke_zh)](https://pepy.tech/project/pke_zh)
[![GitHub contributors](https://img.shields.io/github/contributors/shibing624/pke_zh.svg)](https://github.com/shibing624/pke_zh/graphs/contributors)
[![License Apache 2.0](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![python_vesion](https://img.shields.io/badge/Python-3.5%2B-green.svg)](requirements.txt)
[![GitHub issues](https://img.shields.io/github/issues/shibing624/pke_zh.svg)](https://github.com/shibing624/pke_zh/issues)
[![Wechat Group](http://vlog.sfyc.ltd/wechat_everyday/wxgroup_logo.png?imageView2/0/w/60/h/20)](#Contact)

PKE_zh, Python Keyphrase Extraction for zh(chinese).

**pke_zh**实现了多种中文关键词提取算法，包括有监督的WordRank，无监督的TextRank、TfIdf、KeyBert、PositionRank、TopicRank等，扩展性强，开箱即用。


**Guide**

- [Feature](#Feature)
- [Install](#install)
- [Usage](#usage)
- [Contact](#Contact)
- [Reference](#reference)

# Feature

如何提取query或者文档的关键词？


## 有监督方法
### 特征工程的解决思路
把关键词提取任务转化为分类任务，对输入query句子分词并提取多种特征，再把特征喂给机器学习模型，模型区分出各词的重要性得分，这样挑出topK个词作为关键词。

#### 特征工程

* 文本特征：包括Query长度、Term长度，Term在Query中的偏移量，term词性、长度信息、term数目、位置信息、句法依存tag、是否数字、是否英文、是否停用词、是否专名实体、是否重要行业词、embedding模长、删词差异度、以及短语生成树得到term权重等
* 统计特征：包括PMI、IDF、TextRank值、前后词互信息、左右邻熵、独立检索占比（term单独作为query的qv/所有包含term的query的qv和）、统计概率、idf变种iqf
* 语言模型特征：整个query的语言模型概率 / 去掉该Term后的Query的语言模型概率


训练样本形如：
```shell
邪御天娇 免费 阅读,3 1 1
```

重要度label共分4级：

- Super important：3级，主要包括POI核心词，比如“方特、欢乐谷”
- Required：2级，包括行政区词、品类词等，比如“北京 温泉”中“北京”和“温泉”都很重要
- Important：1级，包括品类词、门票等，比如“顺景 温泉”中“温泉”相对没有那么重要，用户搜“顺景”大部分都是温泉的需求
- Unimportant：0级，包括语气词、代词、泛需求词、停用词等

上例中可见“温泉”在不同的query中重要度是不同的。

分类模型可以是GBDT、LR、SVM、Xgboost等，这里以GBDT为例，GBDT模型（WordRank）的输入是特征向量，输出是重要度label:
![term-weighting](./docs/gbdt.png)

### 深度模型的解决思路
- 思路一：本质依然是把关键词提取任务转化为词重要度分类任务，利用深度模型学习term重要性，取代人工提取特征，模型端到端预测词重要度label，深度模型可以是TextCNN、Fasttext、Transformer等，也可以是BERT预训练模型，适用于分类任务的模型都行。分类任务实现参考：https://github.com/shibing624/pytextclassifier
- 思路二：用Seq2Seq生成模型，基于输入query生成关键词或者摘要，生成模型可以是T5、Bart、Seq2Seq等，生成任务实现参考：https://github.com/shibing624/textgen


#### 分类模型
* TextCNN、FastText、BiLSTM...
* BERT CLS + softmax 

#### 生成模型
* Seq2Seq 文本摘要模型
* T5、Bart、GPT2...

## 无监督方法
- [x] TextRank
- [x] TfIdf
- [x] SingleRank
- [x] PositionRank
- [x] TopicRank
- [x] MultipartiteRank
- [x] Yake
- [x] KeyBert



# Install
* From pip:
```shell
pip install -U pke_zh
```
* From source：
```shell
git clone https://github.com/shibing624/pke_zh.git
cd pke_zh
python setup.py install
```

### 依赖数据
* 千兆中文文本训练的语言模型[zh_giga.no_cna_cmn.prune01244.klm(2.8G)](https://deepspeech.bj.bcebos.com/zh_lm/zh_giga.no_cna_cmn.prune01244.klm)，模型由pycorrector库自动下载于 `~/.pycorrector/datasets/zh_giga.no_cna_cmn.prune01244.klm` 。
* 中文文本匹配模型[shibing624/text2vec-base-chinese](https://huggingface.co/shibing624/text2vec-base-chinese) ，模型由transformers库自动下载于 `~/.cache/huggingface/transformers/` 下。

# Usage

## 有监督关键词提取

直接调用训练好的WordRank模型，模型自动下载于 `~/.cache/pke_zh/wordrank_model.pkl` 。

example: [examples/keyphrase_extraction_demo.py](examples/keyphrase_extraction_demo.py)

```python
from pke_zh.supervised.wordrank import WordRank
m = WordRank()
print(m.extract("哪里下载电视剧周恩来？"))
```

output:
```shell
[('电视剧', 3), ('周恩来', 3), ('下载', 2), ('哪里', 1), ('？', 0)]
```
> 3：核心词；2：限定词；1：可省略词；0：干扰词。

### 基于自有数据训练模型

训练example: [examples/train_supervised_wordrank_demo.py](examples/train_supervised_wordrank_demo.py)


## 无监督关键词提取
支持TextRank、TfIdf、KeyBert等关键词提取算法。

example: [examples/unsupervised_demo.py](examples/unsupervised_demo.py)


```python
from pke_zh.unsupervised.textrank import TextRank
from pke_zh.unsupervised.tfidf import TfIdf
from pke_zh.unsupervised.singlerank import SingleRank
from pke_zh.unsupervised.positionrank import PositionRank
from pke_zh.unsupervised.topicrank import TopicRank
from pke_zh.unsupervised.multipartiterank import MultipartiteRank
from pke_zh.unsupervised.yake import Yake
from pke_zh.unsupervised.keybert import KeyBert

q = '哪里下载电视剧周恩来？'
TextRank_m = TextRank()
TfIdf_m = TfIdf()
PositionRank_m = PositionRank()
KeyBert_m = KeyBert()

r = TextRank_m.extract(q)
print('TextRank:', r)

r = TfIdf_m.extract(q)
print('TfIdf:', r)

r = PositionRank_m.extract(q)
print('PositionRank_m:', r)

r = KeyBert_m.extract(q)
print('KeyBert_m:', r)
```

output:
```shell
TextRank: [('电视剧', 1.00000002)]
TfIdf: [('哪里下载', 1.328307500322222), ('下载电视剧', 1.328307500322222), ('电视剧周恩来', 1.328307500322222)]
PositionRank_m: [('电视剧', 1.0)]
KeyBert_m: [('电视剧', 0.47165293)]
```

### 无监督关键句提取（自动摘要）
支持TextRank摘要提取算法。

example: [examples/keysentences_extraction_demo.py](examples/keysentences_extraction_demo.py)


```python
from pke_zh.unsupervised.textrank import TextRank

m = TextRank()
r = m.extract_sentences("较早进入中国市场的星巴克，是不少小资钟情的品牌。相比 在美国的平民形象，星巴克在中国就显得“高端”得多。用料并无差别的一杯中杯美式咖啡，在美国仅约合人民币12元，国内要卖21元，相当于贵了75%。  第一财经日报")
print(r)
```

output:
```shell
[('相比在美国的平民形象', 0.13208935993025409), ('在美国仅约合人民币12元', 0.1320761453200497), ('星巴克在中国就显得“高端”得多', 0.12497451534612379), ('国内要卖21元', 0.11929080110899569) ...]
```

# Contact

- Issue(建议)：[![GitHub issues](https://img.shields.io/github/issues/shibing624/pke_zh.svg)](https://github.com/shibing624/pke_zh/issues)
- 邮件我：xuming: xuming624@qq.com
- 微信我：加我*微信号：xuming624*, 备注：*姓名-公司名-NLP* 进NLP交流群。

<img src="docs/wechat.jpeg" width="200" />


# Citation

如果你在研究中使用了pke_zh，请按如下格式引用：
APA:
```latex
Xu, M. pke_zh: Python keyphrase extraction toolkit for chinese (Version 0.2.2) [Computer software]. https://github.com/shibing624/pke_zh
```

BibTeX:
```latex
@misc{pke_zh,
  author = {Xu, Ming},
  title = {pke_zh: Python keyphrase extraction toolkit for chinese},
  year = {2023},
  publisher = {GitHub},
  journal = {GitHub repository},
  howpublished = {\url{https://github.com/shibing624/pke_zh}},
}
```

# License


授权协议为 [The Apache License 2.0](LICENSE)，可免费用做商业用途。请在产品说明中附加pke_zh的链接和授权协议。


# Contribute
项目代码还很粗糙，如果大家对代码有所改进，欢迎提交回本项目，在提交之前，注意以下两点：

 - 在`tests`添加相应的单元测试
 - 使用`python -m pytest`来运行所有单元测试，确保所有单测都是通过的

之后即可提交PR。


# Reference

- [boudinfl/pke](https://github.com/boudinfl/pke)
- [Context-Aware Document Term Weighting for Ad-Hoc Search](http://www.cs.cmu.edu/~zhuyund/papers/TheWebConf_2020_Dai.pdf)
- [term weighting](https://zhuanlan.zhihu.com/p/90957854)
- [DeepCT](https://github.com/AdeDZY/DeepCT)
