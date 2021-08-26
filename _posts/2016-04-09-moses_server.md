---
layout: post
title: 基于Moses server的翻译系统
categories: 机器翻译
---

### 前言  

在前面的文章中，我们对moses系统进行了较多了解，如：[图解Moses系统的搭建过程]( http://www.cnblogs.com/panweishadow/p/4771050.html )  
  
现在我们用moses来搭建一个翻译系统，这个系统可以部署在一个单独的服务器上，为网页或者桌面程序提供翻译服务，而这样客户急需的功能恰好是Moses系统自带的。  
  
Moses是能够让我们以server的形式启动进程的，然后通过xmlrpc接收所需要翻译的句子。这也就意味着一个moses进程可以为使用Java，perl，python，php或者其他有xmlrpc库的编程语言编写的客户端(client)提供服务。  
  
**系统结构如下：**  
![翻译系统结构](/images/moses_web_server_arch.png "翻译系统结构")
  
在每个moses服务器上，都运行一个moses.pl的程序，它监听端口，接收用户请求，传递给Moses进行翻译，翻译完成后返回给客户，如果需要同时提供多种语言的翻译服务，我们需要部署多个moses服务器。moses.pl 、index.html、trans_result.php等文件在/moses_decoder/contrib/iSenWeb目录下。  
  
下面，让我们看看如何构建这个的一个系统的详细步骤。  
  
### 安装步骤：  
环境：`Ubuntu 14.04 LTS+PHP+Perl+java`  
  
#### 1.安装Moses  
具体步骤可以参考前文： [图解Moses系统的安装过程](http://www.cnblogs.com/panweishadow/p/4771050.html)，只不过在这里还需要安装`xmlxpc-c`, 然后在编译`Moses`的时候，记得带上编辑参数`”—with-xmlrpc-c=path/to/xmlxpc-c”`.  
  
`example: ~/mosesdecoder$ sudo ./bjam --with-irstlm=/home/panzg/irstlm --with-boost=/home/panzg/boost_1_57_0 --with-cmph=/home/panzg/cmph --with-xmlrpc-c=/home/panzg/xmlrpc-c –j4`  
如果看到`SUCCESS`字样，就编译成功了，查看bin文件夹下是否有`moses server`程序  
![](/images/moses_server_compile_success.png)
  
#### 2.安装PHP和Apache  
只需要一条命令：```udo apt-get install apache2 libapache2-mod-php5```  
  
#### 3.安装Netcat  
只需要一条命令：`sudo apt-get install netcat`  
  
netcat的作用是进行端口监听，请求转发  
  
#### 4.训练、调优Moses  
在这里，你需要使用语料，来对Moses进行训练和调优，具体相关步骤可见：   
  
- [前文](http://www.cnblogs.com/panweishadow/p/4781968.html)   
- [官方手册](http://www.statmt.org/moses/?n=Moses.Baseline)  
  
#### 5.配置moses.pl  
打开moses.pl,修改其中的MOSES和MOSES_INI，将它们指向你的Moses位置和moses.ini位置。如：  
  
- `my $MOSES = '/home/tianliang/research/moses-smt/scripts/training/model/moses';`  
- `my $MOSES_INI = '/home/tianliang/research/moses-smt/scripts/training/model/moses.ini';`   
  
#### 6.运行moses.pl  
为启动moses server,需要运行：  
`./moses.pl <hostname> <port>`  
指定ip和port即可运行，启动成功后，会等待用户的翻译请求.  
  
#### 7.翻译测试  
使用nc工具，发送翻译请求：  
  
![](/images/moses_server_test_example.png)
  
#### 8.网页翻译  
  
将`contrib/iSenWeb`目录下的文件，复制到apache的工作目录下，我的是`/var/www/html`.  
  
注意，这里要修改`trans-result.php`中的内容，指定你机器上的ip和port,然后保存。  
  
查看apache是否启动，在浏览器中输入: localhost即可验证。  
  
在浏览器中输入 `localhost/index.html`，即可进行翻译。  
  
效果如下（en->ch没修改html文件）：  
  
![](/images/moses_server_test_example_2.png)
  
至此，就是一个网页的翻译系统了，该系统的翻译质量取决于你的训练阶段和调优阶段的程度。  
  
#### 9.与桌面程序通信  
  
index.html请求页面通过post方式提交请求：  
  
![](/images/moses_server_post_request.png)
  
因此，我们的程序只需要构造这样的post请求即可，获取进行翻译请求的提交。  
  
下面是java程序模拟post请求，其中httpRequest是封装的http请求类。如下：  
  
![](/images/moses_server_java_demo.png)
  
至此，在我们的程序中也可以使用moses的翻译服务.  
  
#### 10.参考:  
  
- [田亮博客]( http://www.tianliang123.com/post/12)  
- [寒小阳](http://blog.csdn.net/han_xiaoyang/article/details/10109053)  
- [官方手册]( http://www.statmt.org/moses/?n=Moses.WebTranslation)  
  
  
**作者**：[panzg123](http://panzg123.github.io/)  
2016-04-09 由博客园迁移至Github  
若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。 