---
layout: post
title: Mahout Naive Bayes中文新闻分类示例
categories: IR
---
## 一、简介

------------

关于Mahout的介绍，请看这里：[http://mahout.apache.org/](http://mahout.apache.org/ "http://mahout.apache.org/")

关于Naive Bayes的资料，请戳这里：[基于MapReduce的朴素贝叶斯算法的实现与分析](http://www.cnblogs.com/panweishadow/p/4320722.html)

Mahout实现了Naive Bayes分类算法，这里我用它来进行中文的新闻文本分类。

官方有一组分类例子，使用：[20 newsgroups data](http://people.csail.mit.edu/jrennie/20Newsgroups/20news-bydate.tar.gz),总大小约为85MB。

对于中文文本，相比英文文本，只多一步切词的步骤，使用搜狗实验室的语料库，总大小约为300M。请戳这里：[搜狗实验室](http://www.sogou.com/labs/resources.html?v=1)

## 二、详细步骤

1. 写切词小程序，工具包为IK，用空格分开，将所有新闻集中到一个文本中，一行代表一篇新闻~

2. 上传数据到hdfs，数据量大小，亲测数小时~~~
```  
dfs -cp /share/data/Mahout_examples_Data_Set/20news-all .  
```  

3. 从20newsgroups data创建序列文件(sequence files)
```
mahout seqdirectory -i 20news-all -o 20news-seq
```

4. 将序列文件转化为向量
```
mahout seq2sparse -i ./20news-seq -o ./20news-vectors  -lnorm -nv  -wt tfidf
```

5. 将向量数据集分为训练数据和检测数据，以随机40-60拆分
```
mahout split -i ./20news-vectors/tfidf-vectors --trainingOutput ./20news-train-vectors --testOutput ./20news-test-vectors --randomSelectionPct 40 --overwrite --sequenceFiles -xm sequential
```

6. 训练朴素贝叶斯模型
```
mahout trainnb -i  ./20news-train-vectors -el -o ./model -li ./labelindex -ow -c 
```

7. 检验朴素贝叶斯模型
```
mahout testnb -i ./20news-train-vectors -m ./model -l ./labelindex -ow -o 20news-testing –c
```

8. 检测模型分类效果
```
mahout testnb -i ./20news-test-vectors -m ./model -l ./labelindex -ow -o ./20news-testing -c
```

------------

**如果各位觉得这篇博客和代码对您有一定帮助，还请给[本博客](https://github.com/panzhengguang/panzhengguang.github.io)`star`一下，谢谢各位。**

作者：[panzhengguang](https://github.com/panzhengguang)

记录：6-16日迁移自我的博客园[西芒xiaoP](http://www.cnblogs.com/panweishadow/)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。