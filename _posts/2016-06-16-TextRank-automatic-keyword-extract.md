---
layout: post
title: TextRank自动提取文本关键词
categories: IR
---

### 背景

--------

`TextRank`的经典论文，其常用于:

* 抽取文章关键词；
* 抽取摘要；

> Rada M, Tarau P. TextRank: Bringing Order into Texts[C]. Empirical Methods in Natural Language Processing, 2004.

原文下载地址：[TextRank: Bringing Order into Texts](http://web.eecs.umich.edu/~mihalcea/papers/mihalcea.emnlp04.pdf)

这里介绍如何使用TextRank抽取文本关键词，其主要思想是：

* 找到节点（单词、短语、句子）；
* 找出节点关系(句子相似度、词的共现关系)；
* 套用PageRank算法；

如果你对PageRank熟悉，则可以跳过下面一段。

**PageRank**

PageRank最开始用来计算网页的重要性,计算公式如下：

[![](http://7xsvsk.com1.z0.glb.clouddn.com/pagerank_1.png)](http://7xsvsk.com1.z0.glb.clouddn.com/pagerank_1.png)

R(u)和R(v)是分别是网页u、v的PageRank值，Bu指的是指向网页u的网页集合、Nv是网页v的出链数目,d是改善后的随机游走模型中的参数，d通常取0.85。

简单地来说，一个网页的PageRank等于所有的指向它的网页的PageRank的分量之和(c为归一化参数)。

其特性如下(类似于微博的`大V效应`)：

* 一个网页如果它的入链越多，那么它也越重要(PageRank越高)；
* 一个网页如果被越重要的网页所指向，那么它也越重要(PageRank越高) 。

可以证明PageRank公式是收敛的，下面是一个迭代计算的例子。

[![](http://7xsvsk.com1.z0.glb.clouddn.com/pagerank_2.png)](http://7xsvsk.com1.z0.glb.clouddn.com/pagerank_2.png)

### TextRank抽取关键词的实现

-------------------------

原生`TextRank`在`PageRank`的基础上引入了边权重的思想，公式如下：

[![](http://7xsvsk.com1.z0.glb.clouddn.com/textrank_1.png)](http://7xsvsk.com1.z0.glb.clouddn.com/textrank_1.png)

在关键词抽取中，可以简单地把各边的权重简化为1，则上述公式退化为PageRank算法。下面介绍`jieba`中利用TextRank抽取关键词算法的实现。

`jieba`中TextRank抽取关键词的使用demo： [test/demo.py](https://github.com/fxsjy/jieba/blob/master/test/demo.py)

`jieba`用textRank抽取关键词的效果图,文本内容为：

>此外，公司拟对全资子公司吉林欧亚置业有限公司增资4.3亿元，增资后，吉林欧亚置业注册资本由7000万元增加到5亿元。吉林欧亚置业主要经营范围为房地产开发及百货零售等业务。目前在建吉林欧亚城市商业综合体项目。2013年，实现营业收入0万元，实现净利润-139.13万元。

抽取关键词结果：

[![](http://7xsvsk.com1.z0.glb.clouddn.com/textrank_2.png)](http://7xsvsk.com1.z0.glb.clouddn.com/textrank_2.png)

`jieba`的基本思想为：

1.  将待抽取关键词的文本进行分词
2.  以固定窗口大小(默认为5，通过`span`属性调整)，词之间的共现关系，构建图
3.  计算图中节点的PageRank，注意是无向带权图

#### textrank.py源码解析

首先，先读入文本，并切词，对切词结果统计共现关系，窗口大小默认为5，保存到`cm`中.  

```
        cm = defaultdict(int)
        #切词
        words = tuple(self.tokenizer.cut(sentence))
        for i, wp in enumerate(words):
        #过滤词性，停用词等
            if self.pairfilter(wp):
                for j in xrange(i + 1, i + self.span):
                    if j >= len(words):
                        break
                    if not self.pairfilter(words[j]):#过滤
                        continue
                    #保存到字典中
                    if allowPOS and withFlag:
                        cm[(wp, words[j])] += 1
                    else:
                        cm[(wp.word, words[j].word)] += 1
```

接着，构建无向带权图，g=UndirectWeightedGraph，将图结构保存到g中。  

```
  for terms, w in cm.items():
        #遍历cm，构建无向图
        g.addEdge(terms[0], terms[1], w)
```

然后，对无向带权图`g`套用PageRank  

```
def rank(self):
        #ws保存最后的节点值
        ws = defaultdict(float)
        #outSum保存节点的出度
        outSum = defaultdict(float)
        #计算初始值
        wsdef = 1.0 / (len(self.graph) or 1.0)
        #初始化并统计节点出度
        for n, out in self.graph.items():
            ws[n] = wsdef
            outSum[n] = sum((e[2] for e in out), 0.0)
        # this line for build stable iteration
        sorted_keys = sorted(self.graph.keys())
        # 进行10次迭代计算
        for x in xrange(10):
            for n in sorted_keys:
                s = 0
                #套用PageRank公式
                for e in self.graph[n]:
                    s += e[2] / outSum[e[1]] * ws[e[1]]
                # 这里d默认为0.85
                ws[n] = (1 - self.d) + self.d * s
        (min_rank, max_rank) = (sys.float_info[0], sys.float_info[3])
        #最大最小节点值
        for w in itervalues(ws):
            if w < min_rank:
                min_rank = w
            if w > max_rank:
                max_rank = w
        #归一化
        for n, w in ws.items():
            # to unify the weights, don't *100.
            ws[n] = (w - min_rank / 10.0) / (max_rank - min_rank / 10.0)
        return ws
```

最后，对`图g`中节点值排序，值最高的`top k`即为文本的关键词,默认返回20个  

```
if withWeight:
   #排序
   tags = sorted(nodes_rank.items(), key=itemgetter(1), reverse=True)
else:
   tags = sorted(nodes_rank, key=nodes_rank.__getitem__, reverse=True)

#返回top k
if topK:
    return tags[:topK]
else:
    return tags
```

以上就是jieba用TextRank抽取关键词的全过程，mark

### ref
---------

1. [jieba](https://github.com/fxsjy/jieba "jieba")

### 作者信息
------------

作者：[panzhengguang](https://github.com/panzhengguang)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。