---
layout: post
title: 图解Moses系统搭建过程
categories: 机器翻译
---
# 简介

到目前为止，多个开源的SMT系统已经开发出来。其中Moses是由英国爱丁堡大学、德国亚琛工业大学等8家单位联合开发的一个基于短语的统计机器翻译系统。整个系统用C++语言写成，从训练到解码完全开放源代码，可以运行在Linux平台和Windows平台。是目前最流行的基于短语的机器翻译系统。

*   下面将介绍整个环境的安装，我的**配置的环境**为：

1.  Ubuntu 14.04 LTS
2.  g++4.84
3.  boost 1.57
4.  irstlm5.80.08
5.  moses:github上最新版本， [https://github.com/moses-smt/mosesdecoder.git](https://github.com/moses-smt/mosesdecoder.git)
6.  giza：直接git即可，[https://github.com/moses-smt/giza-pp.git](https://github.com/moses-smt/giza-pp.git)
7.  语料： a small (only 130,000 sentences!) data set released for the 2013 Workshop in Machine Translation.

**心得**: 一般来说，直接按照moses的started和baseline安装即可，地址：[http://www.statmt.org/moses/?n=Development.GetStarted](http://www.statmt.org/moses/?n=Development.GetStarted) 和[http://www.statmt.org/moses/?n=Moses.Baseline](http://www.statmt.org/moses/?n=Moses.Baseline) ，但一定要注意 编译Boost,irstlm,Moses 最后要出现SUCCESS的结果，不能出现failed，不然到后面可能还是要重新编译。

如：

[![image](http://images2015.cnblogs.com/blog/442949/201509/442949-20150904165515903-871511286.png "image")](http://images2015.cnblogs.com/blog/442949/201509/442949-20150904165515278-936377105.png)

# 安装依赖

Moses主要依赖于g++和Boost，此外还有其它工具需要安装，如：

<pre style="margin: 0px; padding: 0px; white-space: pre-wrap; word-wrap: break-word;">
   g++
   git
   subversion
   automake
   libtool
   zlib1g-dev
   libboost-all-dev
   libbz2-dev
   liblzma-dev
   python-dev
   graphviz
   imagemagick
   libgoogle-perftools-dev (for tcmalloc)
</pre>

- 注意你的boost尽量比较新，不然可能会出现各种错误。
- 在ubuntu下如果用apt-get install安装，请注意版本问题。

# 编译Boost

编译boost比较简单，下载源码，直接bootstrap.sh, j2 install即可，官方指导手册如下：

这里最好是让boost完全安装，加参数`--buildtype=complete`,虽说慢一点，但避免出问题麻烦。b2是一个boost自带的构建管理工具，如同make等。现在升级为bjam了。

<pre style="margin: 0px; padding: 0px; white-space: pre-wrap; word-wrap: break-word; font-family: 'Courier New' !important; font-size: 12px !important;"> wget http://downloads.sourceforge.net/project/boost/boost/1.55.0/boost_1_55_0.tar.gz
 tar zxvf boost_1_55_0.tar.gz
 cd boost_1_55_0/ ./bootstrap.sh ./b2 -j4 --prefix=$PWD --libdir=$PWD/lib64 --layout=system link=static install || echo FAILURE</pre>

指定安装lib到lib64中，到时候编译moses需要指定boost的目录。
如：`./bjam --with-boost=~/workspace/temp/boost_1_55_0 –j4`
`-j4`是使用4核进行编译，如果你的机器有4核。

# 编译GIZA++

GIZA++是一个统计机器翻译工具，是用来训练IBM模型1-5和HMM词对齐模型的。该软件包还包含了`mkcls`等生成单词训练生成对准模型的工具。使用下面的命令下载并编译GIZA++：

<pre style="margin: 0px; padding: 0px; white-space: pre-wrap; word-wrap: break-word; font-family: 'Courier New' !important; font-size: 12px !important;">git clone https://github.com/moses-smt/giza-pp.git
cd giza-pp 
make</pre>

编译完成后，生成二进制文件：`~/giza-pp/GIZA++-v2/GIZA++`, `~/giza-pp/GIZA++-v2/snt2cooc.out` 和 `~/giza-pp/mkcls-v2/mkcls`.

可以把这些文件在放一起方便使用，反正只要自己清楚位置就行。

# 编译IRSTLM

这里使用`IRSTLM`工具，具体可见网址：[http://sourceforge.net/projects/irstlm/](http://sourceforge.net/projects/irstlm/ "http://sourceforge.net/projects/irstlm/")

我用的5.80.08版本，下载下来后进行编译：

<pre style="margin: 0px; padding: 0px; white-space: pre-wrap; word-wrap: break-word; font-family: 'Courier New' !important; font-size: 12px !important;">tar zxvf irstlm-5.80.03.tgz
cd irstlm-5.80.03 
./regenerate-makefiles.sh 
./configure --prefix=$HOME/irstlm make install</pre>

编译成功后，会出现一些二进制文件和脚本文件，后面要用到build_lm.sh、add-start-end.sh、compile-lm等脚本。

# 编译Moses

从github上下载源代码，为了编译moses请确保依赖都安装好了，这里编译moses最好指定先指定以下语言模型参数：（虽说可以不指定，但免得后面麻烦）

*   `--with-irstlm=/path/to/irstlm`

如果boost指定了安装目录，请用下列参数指定boost位置：

*   `--with-boost=/path/to/boost`

对了另外，在后面二进制化 `binarise the phrase-table` 和 `lexicalised reordering models`的时候，需要用到`CMPH`库，不然会没有`processPhraseTableMin`和`processLexicalTableMin` 两个文件，因此这里需要下载cmph，为moses指定cmph参数：

*   `--with-cmph=/path/to/cmph`

最后指定以下moses安装目录，方便使用：

*   `--prefix=/path/to/prefix`

如果你的机器是多核，可以指定进行多核编译，如4核，提高速度：

*   `-j4`

运行下列命令进行编译，

<pre style="margin: 0px; padding: 0px; white-space: pre-wrap; word-wrap: break-word; font-family: 'Courier New' !important; font-size: 12px !important;">./bjam --with-irstlm=/path/to/irstlm --prefix=/path/to/prefix --with-boost=/path/to/boost --with-cmph=/path/to/cmph –j4</pre>

编译完成后，一定要看到**SUCCESS**字样。。。。。。。

# 运行moses：

下载测试数据，其中包括了配置文件moses.ini，也提供了语言模型，即不需要用自己的irstlm。。。运行下列命令：

<pre style="margin: 0px; padding: 0px; white-space: pre-wrap; word-wrap: break-word; font-family: 'Courier New' !important; font-size: 12px !important;">
 cd ~/mosesdecoder
 wget http://www.statmt.org/moses/download/sample-models.tgz
 tar xzf sample-models.tgz
 cd sample-models</pre>

完成后，运行moses进行解码：

<pre style="margin: 0px; padding: 0px; white-space: pre-wrap; word-wrap: break-word; font-family: 'Courier New' !important; font-size: 12px !important;">cd ~/mosesdecoder/sample-models 
~/mosesdecoder/bin/moses -f phrase-model/moses.ini < phrase-model/in > out</pre>

其中moses编译成功后的二进制解码器，ini是配置文件，in是输入文件（要翻译的句子：`“das ist ein kleines haus”`），out保存输出的翻译句子，

如果一切正常，在out文件中有输入：`this is a small house`

就这样，moses系统安装完成，后续将介绍基于moses的英法和英汉互译。


# 参考：

> 官方：[http://www.statmt.org/moses/?n=Development.GetStarted](http://www.statmt.org/moses/?n=Development.GetStarted "http://www.statmt.org/moses/?n=Development.GetStarted")
> 
> 官方：[http://www.statmt.org/moses/?n=Moses.Baseline](http://www.statmt.org/moses/?n=Moses.Baseline "http://www.statmt.org/moses/?n=Moses.Baseline")
> 
> [寒小阳](http://my.csdn.net/yaoqiang2011):[http://blog.csdn.net/han_xiaoyang/article/details/10101701](http://blog.csdn.net/han_xiaoyang/article/details/10101701 "http://blog.csdn.net/han_xiaoyang/article/details/10101701")
> 
> 一个帮助很大的文档：http://pan.baidu.com/s/1hq3yORE

**      作者**：[西芒xiaoP](http://panzg123.github.io/)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。