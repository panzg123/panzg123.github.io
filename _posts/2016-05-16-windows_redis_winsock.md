---
layout: post
title: redefinition errors in Win32_FDAPI.h
categories: redis
---

最近有需求，程序A需要通过`socket`与程序B通信，接收程序B的数据，然后写入`Redis`，环境是`windows,c++,hiredis,winsock`  

在程序编译过程中碰到一些错误，在此记录下来。  
  
错误如下：  

![错误原因](http://7xsvsk.com1.z0.glb.clouddn.com/hiredis_winsock_err.png)  

出现错误的原因是，`MS`为redis开发`windows`版本的`redis`中，重定义了一些函数，导致了这些错误，如下：  

![hiredis定义](http://7xsvsk.com1.z0.glb.clouddn.com/hiredis_winsock_err2.png "hiredis定义")  

错误解决方法：  

[https://github.com/MSOpenTech/redis/issues/225](https://github.com/MSOpenTech/redis/issues/225)  
  
将包含的`hiredis.h`的代码与包含`winsock2.h`的代码，分开写在两个源代码文件中，然后重新编译即可。    

另外，如果程序多重库链接错误，可能需要强制忽略多余的库链接(此种方法比较不一定适合，但在我的代码中管用)  

![](http://7xsvsk.com1.z0.glb.clouddn.com/hiredis_winsock_err3.png)