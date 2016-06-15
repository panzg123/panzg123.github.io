---
layout: post
title: nginx,memcached,redis网络模型总结
categories: 网络编程
---
### nginx服务器模型

#### nginx进程模型

`nginx`采用多进程模型，含一个`master`进程和多个`worker`进程，worker进程数目可配置，一般与机器CPU核心数目一致，master进程主要职责是：接收外界信号，如star,stop,restart，监控worker进程状态。worker进程主要职责：负责处理客户端请求。

[![nginx进程模型](http://7xsvsk.com1.z0.glb.clouddn.com/nginx_process_model.png "nginx进程模型")](http://7xsvsk.com1.z0.glb.clouddn.com/nginx_process_model.png "nginx进程模型")

>图片来源：http://tengine.taobao.org/book/chapter_02.html#id1

这种进程模型的好处:

1. 每个worker进程相互独立，无需加锁，省掉锁的开销
2. 多个worker进程互相不影响，提高稳定性
3. 多进程提供性能

nginx利用`多进程+非阻塞`的模型，能轻松处理上万连接。其处理请求的大致过程为：

* 一个连接请求过来，worker进程的监听套接字可读（这里涉及到`惊群现象`）
* 处理监听套接字可读事件，accept该连接
* worker进程开始读取请求，解析请求，处理请求，回复数据，断开连接的流程

#### nginx事件处理模型

处理三种常见事件：信号、定时器、网络IO，其中信号有专门的handler来处理，定时器事件和网络IO在主循环中处理。

>用一段伪代码来总结Nginx的事件处理机制：
>来自：http://tengine.taobao.org/book/chapter_02.html

```
while (true) {
    for t in run_tasks:
        t.handler();
    update_time(&now);
    timeout = ETERNITY;
    for t in wait_tasks: /* sorted already 处理定时器事件*/
        if (t.time <= now) {
            t.timeout_handler();
        } else {
            timeout = t.time - now;
            break;
        }
    nevents = poll_function(events, timeout);/*处理网络IO事件*/
    for i in nevents:
        task t;
        if (events[i].type == READ) {
            t.handler = read_handler;
        } else { /* events[i].type == WRITE */
            t.handler = write_handler;
        }
        run_tasks_add(t);
}
```

#### 惊群现象

master进程会事先创建好监听套接字，然后fork worker子进程时，会继承监听套接字，当listen socket可读时，所有进程都将被唤醒，都会去accept这个请求，最终只有一个进程会成功，其他则失败，这就是`惊群现象`

nginx解决方法：提供一个`ngx_accept_mutex`，worker进程尝试accept之前会去获取该mutex，获取成功的采取accept连接。

#### nginx进程间通信

nginx的进程通信分为三种类别：linux 系统与nginx 通信， master 进程与worker进程通信， worker进程间通信。

* linux 系统与nginx之间通信，通过信号进行，通过信号控制nginx重启、关闭以及加载配置文件等。
* master进程与worker进程通信，socket方式，该种方式的优势是，统一封装网络IO事件，循环处理
* worker进程之间通信，共享内存

### Memcached总结

#### 网络模型

memcached是一款服务器缓存软件，基于`libvent`开发，使用的多线程模型，主线程listen\accept，工作线程处理消息。

主要流程是：

1. 创建主线程和若干个工作线程；
2. 主线程监听、接受连接；
3. 然后将连接信息分发（求余）到工作线程，每个工作线程有一个conn_queue处理队列；
4. 工作线程从conn_queue中取连接，处理用户的命令请求并返回数据；

[![memcached网络模型](http://7xsvsk.com1.z0.glb.clouddn.com/memcached_network_model.jpg "memcached网络模型")](http://7xsvsk.com1.z0.glb.clouddn.com/memcached_network_model.jpg "memcached网络模型")

> 图片来源：http://www.lvtao.net/c/623.html

### Redis事件模型

Redis采用`单线程模型`，通过`IO多路复用`来监听多个连接，非阻塞IO，同时单线程避免了不必要的锁的开销。

Redis同时处理文件事件和时间事件

**文件事件**，Redis将产生事件套接字放入一个`队列`中，然后依次分派给文件事件处理器；Redis编写了多个文件事件处理器，如

* 连接应答处理器`networking.c/acceptTcpHandler`；
* 命令请求处理器`networking.c/readQueryFromClinet`；
* 命令回复处理器`networking.c/sendReplyToClient`；

**时间事件**包含`定时事件`和`周期性事件`，Redis将其放入一个`无序链表`中，每当时间事件执行器运行时，就遍历链表，查找已经到达的时间事件，调用相应的处理器。

Redis中一个重要的时间事件是`serverCron`，Redis利用该函数来定期对自身的资源和状态进行检查和调整，时间周期可以参考`redis.conf`关于hz选项说明，其主要工作包含:

1. 更新服务器的各类统计信息
2. 清理数据库中的过期键值对
3. 关闭和清理失效的客户端连接
4. 进行AOF和RDB持久化操作
5. 如果服务器是主服务器，则进行定期同步
6. 如果处于集群模式，对集群定期同步和连接测试

`Redis主函数`关于事件处理的代码表示如下：

```
def main():
    #一直循环处理事件
    while(not_stop){
        aeProcessEvents()
	}
```

下面展示`aeProcessEvents`调度文件事件和时间事件的过程：

```
def aeProcessEvents():
    time_event = aeSearchNearestTimer() #获取当前时间最近的时间事件
	remaind_ms = time_event.when - unix_ts_now() #获取最近的时间事件达到的毫秒时间
	if remaind_ms < 0 : #时间为负数，赋值0
	    remaind_ms = 0
	timeval = create_timeval_with_ms(remainds_ms) #创建等待的时间结构
	aeApiPoll(timeval) #等待文件事件产生，时间取决于remainds_ms
	processFileEvent() #处理文件事件
	processTimeEvent() #处理时间事件
```

### Libevent网络模型总结

占坑

### Reference
1. [Nginx从入门到精通](http://tengine.taobao.org/book/chapter_02.html "Nginx从入门到精通")
2. [lvtao博客](http://www.lvtao.net/c/623.html)
3. [Redis设计与实现](http://item.jd.com/11486101.html)