---
layout: post
title: Nginx安装、性能测试、反向代理、负载均衡实例
categories: 网络编程
---

# 一、nginx安装

我使用的环境是64位 Ubuntu 14.04。nginx依赖以下模块：

1. gzip模块需要 zlib 库

1. rewrite模块需要 pcre 库

1. ssl 功能需要openssl库

## 1.1.安装pcre

1. 获取pcre编译安装包，在http://www.pcre.org/ 上可以获取当前最新的版本

1. 解压缩pcre-xx.tar.gz包。

1. 进入解压缩目录，执行./configure。

1. make & make install

## 1.2.安**装openssl**

1. 获取openssl编译安装包，在http://www.openssl.org/source/ 上可以获取当前最新的版本。

1. 解压缩openssl-xx.tar.gz包。

1. 进入解压缩目录，执行./config。

1. make & make install

## 1.3.安装zlib

1. 获取zlib编译安装包，在http://www.zlib.net/ 上可以获取当前最新的版本。

1. 解压缩openssl-xx.tar.gz包。

1. 进入解压缩目录，执行./configure。

1. make & make install

## 1.4.安装nginx

1. 获取nginx，在http://nginx.org/en/download.html 上可以获取当前最新的版本。

1. 解压缩nginx-xx.tar.gz包。

1. 进入解压缩目录，执行./configure

1. make & make install

1. 或者省略以上步骤，直接sudo apt-get install nginx (配置和安装文件的位置/usr/share/nginx , /etc/nginx , /etc/default/nginx)

若安装时找不到上述依赖模块，使用--with-openssl=<openssl_dir>、--with-pcre=<pcre_dir>、--with-zlib=<zlib_dir>指定依赖的模块目录。如已安装过，此处的路径为安装目录；若未安装，则此路径为编译安装包路径，nginx将执行模块的默认编译安装。

启动nginx之后，浏览器中输入http://localhost 可以验证是否安装启动成功。

[![clip_image001](http://images0.cnblogs.com/blog/442949/201505/300917241889933.jpg "clip_image001")](http://images0.cnblogs.com/blog/442949/201505/300917236574605.jpg)

# 二、nginx配置

1\. nginx.conf主配置文件

2\. mime.types文件扩展名与文件类型映射表

nginx根据映射关系，设置http请求响应头的Content-Type值。

3\. fastcgi_params

nginx配置Fastcgi解析时会调用fastcgi_params配置文件来传递服务器变量，这样CGI中可以获取到这些变量的值。

4\. fastcgi.conf

资源：http://www.cnblogs.com/xiaogangqq123/archive/2011/03/02/1969006.html

# 三、性能测试demo

1\. 使用陈硕的测试方法，参见《C++多线程服务器编程》。

配置文件的写法：

```c
user www-data;

worker_processes 1;

pid /var/run/nginx.pid;

events {

worker_connections 1024;

# multi_accept on;

}

http {

include mime.types;

default-type application/octet-stream;

access_log off;

senfile on;

tcp_nopush on;

keppalive_timeout 65;

server{

listen 8080;

server_name localhost;

location / {

root html;

index index.html index.htm;

}

location /hello{

default_type text/plain;

echo "hello,world!";

}

}

}
```

2\. 使用ab进行测试：

ab的安装方法：sudo apt-get install apache-utils2

ab –n 10000 –c 1000 localhost:8080/hello

访问数1w，并发数目1k。

资源：http://www.cnblogs.com/yjf512/archive/2011/05/24/2055723.html

3\. 测试结果：

[![clip_image003](http://images0.cnblogs.com/blog/442949/201505/300917251577863.jpg "clip_image003")](http://images0.cnblogs.com/blog/442949/201505/300917246888020.jpg)

# 四、反向代理缓存+负载均衡，还待写

环境：vm+CentOs；Nginx、MySQL、PHP、Apache

## 1.反向代理缓存

配置方法：

效果：

## 2.负载均衡

配置方法：

效果：


------------


**作者**：[西芒xiaoP](http://www.cnblogs.com/panweishadow/)

**出处**：迁移自博客[`西芒xiaoP`](http://www.cnblogs.com/panweishadow/)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。