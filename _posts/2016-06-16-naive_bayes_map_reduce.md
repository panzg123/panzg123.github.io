---
layout: post
title: 基于MapReduce的朴素贝叶斯算法的实现与分析
categories: IR
---
## 一、朴素贝叶斯(Naïve Bayes)分类器

--------------------------------------

### 1.1 公式

`朴素贝叶斯`是一个概率分类器

文档 d 属于类别 c 的概率计算如下（多项式模型）：

> ![image](http://images.cnitblog.com/blog/442949/201503/071744167112138.png "image")

*  ` nd`是文档的长度(词条的个数)
*   `P(tk |c)` 是词项tk 出现在类别c中文档的概率，即类别c文档的一元语言模型
*   `P(tk |c)` 度量的是当c是正确类别时tk 的贡献
*   `P(c)` 是类别c的先验概率

如果文档的词项无法提供属于哪个类别的信息，那么我们直接选择P(c)最高的那个类别

### 1.2 具有最大后验概率的类别

朴素贝叶斯分类的目标是寻找“最佳”的类别

最佳类别是指具有最大后验概率`(maximum a posteriori -MAP)`的类别 cmap:

![image](http://images.cnitblog.com/blog/442949/201503/071744175244552.png "image")

### 1.3 对数计算 

很多小概率的乘积会导致浮点数下溢出

由于 `log(xy) = log(x) + log(y)`, 可以通过取对数将原来的乘积计算变成求和计算

由于log是单调函数，因此得分最高的类别不会发生改变

因此，实际中常常使用的是：

![image](http://images.cnitblog.com/blog/442949/201503/071744180242638.png "image")

### 1.4 零概率问题

如果某词项不再某类别中出现，则包含词项的文档属于该类别的概率P=0,即一旦发生令概率，则无法判断类别。

解决方法：`加一平滑`。

平滑前：

[![image](http://images.cnitblog.com/blog/442949/201503/071744192277339.png "image")](http://images.cnitblog.com/blog/442949/201503/071744186648468.png)

平滑后: 对每个量都加上1

[![image](http://images.cnitblog.com/blog/442949/201503/071744207745611.png "image")](http://images.cnitblog.com/blog/442949/201503/071744201173011.png)

B 是不同的词语个数 (这种情况下词汇表大小 |V | = B)

### 1.5 两种常见模型

这里需要提到贝叶斯模型的两个独立性假设：位置独立性和条件独立性。

`多项式模型`和`贝努力模型`。前者考虑出现次数，后者只考虑出现与不出现，即0 与 1问题。

### 1.6 算法过程

`训练过程：`

[![image](http://images.cnitblog.com/blog/442949/201503/071744225086642.png "image")](http://images.cnitblog.com/blog/442949/201503/071744214616456.png)

`测试/应用/分类：`

![image](http://images.cnitblog.com/blog/442949/201503/071744232588271.png "image")

### 1.7 实例

[![image](http://images.cnitblog.com/blog/442949/201503/071744247899772.png "image")](http://images.cnitblog.com/blog/442949/201503/071744240246671.png)

**首先，第一步，参数估计：**

[![image](http://images.cnitblog.com/blog/442949/201503/071744264927858.png "image")](http://images.cnitblog.com/blog/442949/201503/071744256178957.png)

**接着，第二步，分类：**

[![image](http://images.cnitblog.com/blog/442949/201503/071744284142647.png "image")](http://images.cnitblog.com/blog/442949/201503/071744274924030.png) 

因此, 分类器将测试文档分到c = `China类`，这是因为d5中起正向作用的CHINESE出现3次的权重高于起反向作用的 JAPAN和TOKYO的权重之和。

## 二、基于MR的并行化实现

----------------------

分两个阶段进行，首先是训练获取分类器，然后是预测。

文件的输入格式：每一行代表一个文本，`格式为：类别名+文件名+文本内容`

[![image](http://images.cnitblog.com/blog/442949/201503/071744305391263.png "image")](http://images.cnitblog.com/blog/442949/201503/071744294304590.png)

### 2.1 训练阶段

训练阶段要计算两种概率：[1] 每种类别的先验概率 ; [2] 每个term（单词）在每个类别中的条件概率。

这里采用多项式模型计算条件概率。

这里有两个`job`完成，伪代码如下：

> 这两段代码来自dongxicheng的博客，原文地址[http://dongxicheng.org/data-mining/naive-bayes-in-hadoop/](http://dongxicheng.org/data-mining/naive-bayes-in-hadoop/ "http://dongxicheng.org/data-mining/naive-bayes-in-hadoop/")

[![image](http://images.cnitblog.com/blog/442949/201503/071744339779378.png "image")](http://images.cnitblog.com/blog/442949/201503/071744328673706.png) 

[![image](http://images.cnitblog.com/blog/442949/201503/071744371332410.png "image")](http://images.cnitblog.com/blog/442949/201503/071744358832695.png)

### 2.2 测试阶段

将训练阶段得到的数据加载到内存中，计算文档在每个类别的概率，找出概率最大的类别。

[![image](http://images.cnitblog.com/blog/442949/201503/071744421023028.png "image")](http://images.cnitblog.com/blog/442949/201503/071744411809112.png)

## 三、MR分析
---------------------
测试数据：搜狗实验室 [http://www.sogou.com/labs/resources.html?v=1](http://www.sogou.com/labs/resources.html?v=1 "http://www.sogou.com/labs/resources.html?v=1")

这里要先要将所有文档转成需要的文本格式，即一行代表一篇新闻。

训练集：75000篇新闻；测试集：5000篇新闻

最终测得分类精度为82%。

------------

**如果各位觉得这篇博客和代码对您有一定帮助，还请给[本博客](https://github.com/panzg123/panzg123.github.io)`star`一下，谢谢各位。**

作者：[panzg123](https://github.com/panzg123)

记录：6-16日迁移自我的博客园[西芒xiaoP](http://www.cnblogs.com/panweishadow/)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。