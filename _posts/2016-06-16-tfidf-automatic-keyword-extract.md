---
layout: post
title: tf-idf自动提取文本关键词
categories: IR
---
### 背景

先上一个链接：[TF-IDF与余弦相似性的应用（一）：自动提取关键词](http://www.ruanyifeng.com/blog/2013/03/tf-idf.html "TF-IDF与余弦相似性的应用（一）：自动提取关键词")

**词频TF**

词项t的词项频率(以下简称词频),是指t在d中出现的次数，是与文档相关的一个量,反应局部信息。

**逆文档频率IDF**

通常，文档频率(document frequency, df)指的是出现词项的文档数目，那么逆文档频率（Inverse Document Frequency，idf）则是根据词项出现的普遍程度来反应词项的重要程度，是反映词项t的信息量的一个指标，是一种全局性指标。

计算公式为：

[![](http://7xsvsk.com1.z0.glb.clouddn.com/idf.jpg)](http://7xsvsk.com1.z0.glb.clouddn.com/idf.jpg)

**TF-IDF计算**

计算公式，信息检索中最出名的权重计算方法：

[![](http://7xsvsk.com1.z0.glb.clouddn.com/tfidf.jpg)](http://7xsvsk.com1.z0.glb.clouddn.com/tfidf.jpg)

可见tf-idf权重，着词项频率的增大而增大; 随着词项罕见度的增加而增大。

以上，自动提取关键词的算法，就是计算出文档的每个词的TF-IDF值，然后按降序排列，取`top k`即可。

---------------------

### 开源工具包

`jieba`提供两种算法的文本关键词提取功能：

1. 基于`TF-IDF`算法的关键词提取
2. 基于`TextRank`算法的关键词提取

这里只介绍方法1，方法1`demo`,关键词提取，[https://github.com/fxsjy/jieba/blob/master/test/extract_tags.py](https://github.com/fxsjy/jieba/blob/master/test/extract_tags.py)

其他接口:

* ` jieba.analyse.set_idf_path(file_name)`自定义idf文件
* ` jieba.analyse.set_stop_words(file_name)`自定义停用词文件。

------------------

### 算法实现

主要参考`jieba`中`tf-idf`提取关键词的实现

算法流程如下：

1. 加载idf文件到内存中，获得每个单词的idf值，如单词未出现，则取中位数
2. 加载要提取关键词的文本，将文本切词，用jieba的切词功能
3. 统计每个词的词频，即`tf`
4. 计算每个词的`tf-idf`值，返回`top k`,k为约定的关键词的数目

Note：idf值需要经过大语料库来统计获取，这样抽取关键词的效果才更好，idf文件样本数据如下：

`格式：词项 idf值`

[![](http://7xsvsk.com1.z0.glb.clouddn.com/idf_txt.png)](http://7xsvsk.com1.z0.glb.clouddn.com/idf_txt.png)

**tfidf.py源码注解：**

```
class KeywordExtractor(object):
    #默认的停用词
    STOP_WORDS = set((
        "the", "of", "is", "and", "to", "in", "that", "we", "for", "an", "are",
        "by", "be", "as", "on", "with", "can", "if", "from", "which", "you", "it",
        "this", "then", "at", "have", "all", "not", "one", "has", "or", "that"
    ))
    #加载用户自定义停用词文件
    def set_stop_words(self, stop_words_path):
        abs_path = _get_abs_path(stop_words_path)
        if not os.path.isfile(abs_path):
            raise Exception("jieba: file does not exist: " + abs_path)
        content = open(abs_path, 'rb').read().decode('utf-8')
        #保存到stop_words中
        for line in content.splitlines():
            self.stop_words.add(line)

    def extract_tags(self, *args, **kwargs):
        raise NotImplementedError

#idf文件加载类
class IDFLoader(object):

    def __init__(self, idf_path=None):
        self.path = ""
        self.idf_freq = {}
        self.median_idf = 0.0
        if idf_path:
            self.set_new_path(idf_path)
    
    #从idf文件中读取词的idf值，保存到idf_freq字典中，用median_idf记录中位数
    def set_new_path(self, new_idf_path):
        if self.path != new_idf_path:
            self.path = new_idf_path
            content = open(new_idf_path, 'rb').read().decode('utf-8')
            self.idf_freq = {}
            for line in content.splitlines():
                word, freq = line.strip().split(' ')
                #保存词的idf值
                self.idf_freq[word] = float(freq)
            #中位数
            self.median_idf = sorted(
                self.idf_freq.values())[len(self.idf_freq) // 2]
    #获取idf值，median值
    def get_idf(self):
        return self.idf_freq, self.median_idf


class TFIDF(KeywordExtractor):

    def __init__(self, idf_path=None):
        self.tokenizer = jieba.dt #分词器
        self.postokenizer = jieba.posseg.dt
        self.stop_words = self.STOP_WORDS.copy() #停用词
        self.idf_loader = IDFLoader(idf_path or DEFAULT_IDF) #idf加载器
        self.idf_freq, self.median_idf = self.idf_loader.get_idf() #idf值字典
    #设置自定义路径
    def set_idf_path(self, idf_path):
        new_abs_path = _get_abs_path(idf_path)
        if not os.path.isfile(new_abs_path):
            raise Exception("jieba: file does not exist: " + new_abs_path)
        self.idf_loader.set_new_path(new_abs_path)
        self.idf_freq, self.median_idf = self.idf_loader.get_idf()

    def extract_tags(self, sentence, topK=20, withWeight=False, allowPOS=(), withFlag=False):
        """
        Extract keywords from sentence using TF-IDF algorithm.
        Parameter:
            - topK: 关键词数目. `None`代表全部
            - withWeight: 为真返回(word,weight)，为假只返回word
            - allowPOS: 关键词的词性列表，不在此列表的将被过滤
            - withFlag: 
        """
        #开始切词
        if allowPOS:
            allowPOS = frozenset(allowPOS)
            words = self.postokenizer.cut(sentence)
        else:
            words = self.tokenizer.cut(sentence)
        freq = {}
        for w in words:
            if allowPOS:
                if w.flag not in allowPOS:
                    continue
                elif not withFlag:
                    w = w.word
            wc = w.word if allowPOS and withFlag else w
            #停用词过滤
            if len(wc.strip()) < 2 or wc.lower() in self.stop_words:
                continue
            #计算tf值
            freq[w] = freq.get(w, 0.0) + 1.0
        total = sum(freq.values())
        for k in freq:
            kw = k.word if allowPOS and withFlag else k
            #计算tf_idf值
            freq[k] *= self.idf_freq.get(kw, self.median_idf) / total
        #返回tf-idf值最高的k个，作为关键词
        if withWeight:
            tags = sorted(freq.items(), key=itemgetter(1), reverse=True)
        else:
            tags = sorted(freq, key=freq.__getitem__, reverse=True)
        if topK:
            return tags[:topK]
        else:
            return tags
```

### ref

1. [jieba](https://github.com/fxsjy/jieba "jieba")
2. [TF-IDF与余弦相似性的应用（一）：自动提取关键词](http://www.ruanyifeng.com/blog/2013/03/tf-idf.html)

------------

作者：[panzhengguang](https://github.com/panzhengguang)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。