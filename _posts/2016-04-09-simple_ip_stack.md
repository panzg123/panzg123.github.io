---
layout: post
title: 一个简单网络协议栈的实现
categories: 网络编程
---
# 1.结构图

[![image](http://images0.cnblogs.com/blog/442949/201506/120005155512780.png "image")](http://images0.cnblogs.com/blog/442949/201506/120005145359424.png)

# 2.程序功能：

该网络协议栈主要包含如下几个部分的协议的支持：

*   以太网的支持
*   IP协议的支持
*   ICMP协议的支持
*   UDP协议的支持
*   协议抽象层的支持
*   用户接口的支持

# 3.源码结构图

源代码地址：[https://github.com/panzg123/Unix_Net_Programming/tree/master/SimpleStack](https://github.com/panzg123/Unix_Net_Programming/tree/master/SimpleStack "https://github.com/panzg123/Unix_Net_Programming/tree/master/SimpleStack")
  
```  
src  
|---makefile  
|---sip.h   
|---sip.c     “主程序”  
|---sip_arp.h  “arp协议”  
|---sip_arp.c  
|---sip_ether.h  “以太网”
|---sip_ether.c
|---sip_icmp.h   “ICMP协议”
|---sip_icmp.c
|---sip_igmp.h    “IGMP协议”
|---sip_igmp.c
|---sip_sock.h   “sock操作”
|---sip_sock.c
|---sip_socket.h   “ 应用层接口”
|---sip_socket.c
|---sip_udp.h      “udp协议”
|---sip_udp.c
|---sip_skbuff.h   “消息缓冲区”
```

# 4.参考

> **《Linux网络编程》**

 　　**作者**：[西芒xiaoP & panzg123](https://github.com/panzg123)

**　　出处**：同步自博客[`西芒xiaoP`](http://www.cnblogs.com/panweishadow/)

　　若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。