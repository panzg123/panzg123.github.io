---
layout: post
title: redis中事件模型实现分析
categories: redis
---
### 1. 模型结构

`Redis`没有使用第三方的libevent等网络库，而是自己开发了一个单线程的Reactor模型的事件处理模型。而`Memcached`内部使用的libevent库，多线程模型。

综合对比可见：**[nginx,memcached,redis网络模型总结](http://panzg123.github.io/2016/06/15/nginx_memcached_server_model/)**

Redis在主循环中统一处理文件事件和时间事件，信号事件则由专门的handler来处理。

**文件事件**，我理解为IO事件，Redis将产生事件套接字放入一个就绪队列中，即**redisServer.aeEventLoop.fired数组**，然后在`aeProcessEvents`会依次分派给文件事件处理器；

Redis编写了多个文件事件处理器。

Redis中文件事件包括：客户端的连接、命令请求、数据回复、连接断开，当上述事件发生时，会造成相应的描述符可读可写，再调用相应类型的文件事件处理器。

文件事件处理器有：

*   连接应答处理器`networking.c/acceptTcpHandler`；
*   命令请求处理器`networking.c/readQueryFromClinet`；
*   命令回复处理器`networking.c/sendReplyToClient`；

**时间事件**包含`定时事件`和`周期性事件`，Redis将其放入一个无序链表中，每当时间事件执行器运行时，就遍历链表，查找已经到达的时间事件，调用相应的处理器。

**主循环**

```
def ae_Main():
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

### 2. 文件事件的处理

现存多种IO复用方法，比如`select`,`poll`,`epoll`,`kqueue`等，每种方法的效率和使用方法都不相同，Redis通过统一包装各方法，来屏蔽它们的不同之处。

**2.1 IO复用跨平台**

首先，Redis会根据平台，自动选择性能最好的IO复用函数库。该过程提现在`Ae.c`头文件包含中，如下：

```
#ifdef HAVE_EVPORT
#include "ae_evport.c" //evport优先级最高
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c" //epoll优先级较次
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c" //kqueue优先级还次
        #else
        #include "ae_select.c" //select优先级最低
        #endif
    #endif
#endif
```
注意：这里`include "xxx.c"`文件

**疑问？** windows只支持上面的select，是否windows平台下redis性能很低？


**2.2 事件接口**

`ae_select.c`、`ae_epoll.c`、`ae_kqueue.c`、`ae_evport.c`都提供一套统一的事件注册、删除接口，使得在`ae.c`中可以直接使用以下接口：

1. `aeApiCreate`创建实例
2. `aeApiResize`
3. `aeApiFree`
4. `aeApiAddEvent`注册事件
5. `aeApiDelEvent`删除时间
6. `aeApiPoll` 获取就绪事件

Redis在调用`InitServer`初始化服务器时，会创建一个`aeEventLoop`结构体，该结构体记录事件处理器的状态，保存了注册的事件和相应的处理器。每个`redisServer`实例都有一个`aeEventLoop`结构体。

```

/* State of an event based program 
 * 事件处理器的状态
 */
typedef struct aeEventLoop {
    // 目前已注册的最大描述符
    int maxfd;   /* highest file descriptor currently registered */
    // 目前已追踪的最大描述符
    int setsize; /* max number of file descriptors tracked */
    // 用于生成时间事件 id
    long long timeEventNextId;
    // 最后一次执行时间事件的时间
    time_t lastTime;     /* Used to detect system clock skew */
    // 已注册的文件事件
    aeFileEvent *events; /* Registered events */
    // 已就绪的文件事件
    aeFiredEvent *fired; /* Fired events */
    // 时间事件
    aeTimeEvent *timeEventHead;
    // 事件处理器的开关
    int stop;
    // 多路复用库的私有数据
    void *apidata; /* This is used for polling API specific data */
    // 在处理事件前要执行的函数
    aeBeforeSleepProc *beforesleep;
} aeEventLoop;
```

`ae.c`中提供了`aeCreateFileEvent`、`aeDeleteFileEvent`等接口，来注册感兴趣的事件。

例子：注册文件的读写事件`aeCreateFileEvent`，

```
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    if (fd >= eventLoop->setsize) return AE_ERR;
    // 取出文件事件结构
    aeFileEvent *fe = &eventLoop->events[fd];
    // 监听指定 fd 的指定事件
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    // 设置文件事件类型，以及事件的处理器
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    // 私有数据
    fe->clientData = clientData;
    // 如果有需要，更新事件处理器的最大 fd
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```

**2.3 文件事件处理器的关联过程：**

1. **连接请求**`acceptTcpHandler`：在 `redis.c/initServer`中，程序会为`redisServer.eventLoop`关联一个客户连接的事件处理器。
2. **命令请求**`readQueryFromClinet`:当新连接来的时候，需要调用`networking.c/createClient`创建客户端，在其中为客户端套接字注册读事件，关联处理器`readQueryFromClinet`。
3. **命令回复**`sendReplyToClient`:当Redis调用`networking.c/addReply`时，会调用`prepareClientToWrite`来注册写事件，当套接字可写时，触发`sendReplyToClient`发送命令回复。

### 3 时间事件

**3.1 背景**

Redis中时间事件主要有两类：

1. 定时事件：某程序在指定时间后执行
2. 周期性事件：某程序每间隔指定时间就执行一次。

Redis时间事件有`aeTimerEvent`结构体来表示，主要包括如下成员：

+ id,事件标识符
+ when_ms,事件到达时间
+ timeProc，时间事件处理器
+ next指针，指向下一个时间事件

那么Redis如何区分定时事件和周期性事件呢？
答：通过事件处理器的返回值来确定，如果返回`ae.h/AE_NOMORE`即-1，则为定时事件，否则为周期性事件，比如`serverCron`返回的是`return 1000/server.hz;`

**3.2 时间事件的注册**

Redis通过`aeCreateTimerEvent`来创建时间事件并注册，就是将该事件放在`redisServer.eventLoop`的时间链表头部，即赋值给**redisServer.eventLoop.timeEventHead**指针。

该过程的源码如下：

```
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc)
{
    // 更新时间计数器
    long long id = eventLoop->timeEventNextId++;
    // 创建时间事件结构
    aeTimeEvent *te;
    te = zmalloc(sizeof(*te));
    if (te == NULL) return AE_ERR;
    // 设置 ID
    te->id = id;
    // 设定处理事件的时间
    aeAddMillisecondsToNow(milliseconds,&te->when_sec,&te->when_ms);
    // 设置事件处理器
    te->timeProc = proc;
    te->finalizerProc = finalizerProc;
    // 设置私有数据
    te->clientData = clientData;
    // 将新事件放入链表表头，这一步很重要
    te->next = eventLoop->timeEventHead;
    eventLoop->timeEventHead = te;
    return id;
}
```

时间事件通过链表保存的，该链表不是按照时间排序的，新插入的时间事件总在头部。因此，在获取最近的时间事件时(`aeProcessEvents`中需要获得等待时间)，我们需要遍历整个链表结构。如**aeSearchNearestTimer**所示:

```
// 寻找里目前时间最近的时间事件
// 因为链表是乱序的，所以查找复杂度为 O（N）
static aeTimeEvent *aeSearchNearestTimer(aeEventLoop *eventLoop)
{
    aeTimeEvent *te = eventLoop->timeEventHead;
    aeTimeEvent *nearest = NULL;
    //遍历链表，找时间最小值
    while(te) {
        if (!nearest || te->when_sec < nearest->when_sec ||
                (te->when_sec == nearest->when_sec &&
                 te->when_ms < nearest->when_ms))
            nearest = te;
        te = te->next;
    }
    return nearest;
}
```

目前**Redis**中，我只发现一个`serverCron`周期性事件，其余的时间事件没发现。serverCron在**initServer**被注册.

注：在**Benchmark**模式下，会注册一个`showThroughput`周期性事件。

```
void initServer()
{
    ..................省略
	 // 为 serverCron() 创建时间事件
    if(aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        redisPanic("Can't create the serverCron time event.");
        exit(1);
    }
	.................省略
}
```
**serverCron函数作用**：定期对服务器自身的状态进行检查和调整，

+ 更新服务器的各类统计信息，比如时间、内存占用
+ 清理数据库过期键值
+ 关闭和清理失效连接
+ 尝试进行AOF或者RDB
+ 如果是主服务器，则定义对从服务器同步
+ 集群模式，则定期同步和连接测试

**3.3 时间事件的处理**

在`aeMain`主循环中，通过层层调用，不断循环的通过**processTimeEvents**来处理链表上的到期时间事件，整个过程很简单：遍历`aeEventLoop.timeEventHead`链表，获取当前时钟，检查是否到期，到期调用`te->timeProc`执行事件处理器，通过返回值retVal判断是否周期性事件，不是则需要删除该事件；

**processTimeEvent**源码参考：

```
/*
 * 处理所有已到达的时间事件
 */
static int processTimeEvents(aeEventLoop *eventLoop) {
//............省略
    // 遍历链表
    // 执行那些已经到达的事件
    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    while(te) {
	    //............
        // 获取当前时间
        aeGetTime(&now_sec, &now_ms);
        // 如果当前时间等于或等于事件的执行时间，那么说明事件已到达，执行这个事件
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;
            id = te->id;
            // 执行事件处理器，并获取返回值
            retval = te->timeProc(eventLoop, id, te->clientData);
            processed++;
            // 记录是否有需要循环执行这个事件时间
            if (retval != AE_NOMORE) {
                // 是的， retval 毫秒之后继续执行这个时间事件
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                // 不，将这个事件删除
                aeDeleteTimeEvent(eventLoop, id);
            }
            // 因为执行事件之后，事件列表可能已经被改变了
            // 因此需要将 te 放回表头，继续开始执行事件
            te = eventLoop->timeEventHead;
        } else {
            te = te->next;
        }
    }
    return processed;
}
```

### 4 事件循环调度

Redis在ae.c中的**aeMain**中循环处理事件，aeMain不断的循环调用**aeProcessEvents**来处理文件事件和时间事件。

`aeProcessEvents`的处理流程，关键部分如下：

```
int aeProcessEvents(aeEventLoop *eventLoop, int flags){
     ....省略
        // 获取就绪文件事件，阻塞时间由最近的时间事件决定
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            // 从已就绪数组中获取事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            // 获取参数
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

            // 处理读事件
            if (fe->mask & mask & AE_READABLE) {
                // rfired 确保读/写事件只能执行其中一个
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            // 处理写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;
        }
    }
    // 处理时间事件
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);
}
```

**整个Redis服务器的流程就可以概括为：**

```
int main()
{
    initServer(); //初始化服务器，读取配置文件，注册监听事件和周期性事件，读取AOF或者RDB
	aeMain();//事件循环，处理文件事件和时间事件
	aeDeleteEventLoop()//上面的aeMain循环跳出，代表服务器需要关闭
}
```

### 5 信号的处理

**Redis**会`initServer`函数中注册信号处理函数，忽略SIGHUP、SIGPIPE信号，为`SIGTERM`信号添加处理函数，如果Redis在Linux、Apple平台，则同时会添加SIGSEGV、SIGBUS、SIGFPE、SIGILL信号。


**Redis中信号注册函数：**

```
void setupSignalHandlers(void) {
    struct sigaction act;
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
    act.sa_handler = sigtermHandler; //注册SIGTERM信号处理函数sigtermHandler
    sigaction(SIGTERM, &act, NULL);

#ifdef HAVE_BACKTRACE //如果定义了HAVE_BACKTRACE
    sigemptyset(&act.sa_mask);
    act.sa_flags = SA_NODEFER | SA_RESETHAND | SA_SIGINFO;
    act.sa_sigaction = sigsegvHandler; //注册SIGSEGV,SIGBUS等信号处理函数sigsegvHandler
    sigaction(SIGSEGV, &act, NULL);
    sigaction(SIGBUS, &act, NULL);
    sigaction(SIGFPE, &act, NULL);
    sigaction(SIGILL, &act, NULL);
#endif
    return;
}
```

附一下各个信号的产生场景：

+ **SIGHUP** 异常断开
+ **SIGPIPE** 管道异常
+ **SIGTERM** 程序的终止信号
+ **SIGSEGV** 内存的非法访问
+ **SIGBUS** 非法地址
+ **SIGFPE** 算术运算致命错误
+ **SIGILL** 非法指令

### 6 Rerference

1.  黄健宏. Redis设计与实现[M]. 机械工业出版社, 2014.


--------------------------------------------

**如果各位觉得这篇博客和代码对您有一定帮助，还请给[本博客](https://github.com/panzg123/panzg123.github.io)`star`一下，谢谢各位。**

作者：[panzg123](https://github.com/panzg123)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。