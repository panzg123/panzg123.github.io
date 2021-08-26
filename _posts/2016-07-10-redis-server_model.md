---
layout: post
title: redis中服务器流程分析
categories: redis
---

### 1 背景

前面总结了redis中数据库的键空间和redis中的事件模型，本节总结下Redis如何处理命令？以及Redis中重要的`serverCron`周期性事件。

本篇主要介绍几面几个点：

1. Redis初始化过程
2. 连接请求、处理命令、发送回复的流程
3. serverCron周期性事件的内部流程

### 2 initServer初始化服务器

服务器初始化过程主要有以下几步：

1. 初始化redisServer结构体
2. 读取配置信息
3. 为相应的数据结构分配空间
4. 读取AOF或者RDB恢复数据库状态
5. aeMain事件循环

**2.1 redisServer结构体**

每个redis实例都由一个redisServer结构体来表示，其包含众多属性，而首先就需要初始化相关的属性。

在main主函数中调用`initServerConfig`来完成这部分工作。

主要内容有：

+ 设置服务器运行ID
+ 设置配置文件路径
+ 设置serverCron的频率
+ 设置端口号
+ 设置位长，32 or 64
+ 设置监听套接字属性，比如监听队列长度、keepalive选项
+ 设置AOF和RDB属性
+ 设置ziplist,intset等适用条件
+ 初始化LRU时钟
+ 创建命令表

`initServerConfig`执行结束之后，会判断是否为`sentinel`模式，如果是，只需要初始化sentinel相关选项。

**2.2 redis config**

接着首先读取`redis-server`的命令参数，比如`--port 6379`，或者在命令行中指定redis.conf文件。

接下来，调用`loadServerConfig`来读取配置文件，读取到的信息保存到一个sds字符串中，然后将sds字符串传递给`config.c/loadServerConfigFromString`来配置。

我们使用redis时，一般在redis.conf中配置相关选项，比如pid路径、日志文件、端口、AOF和RDB机制、最大内存等数据。

**2.3 数据结构分配空间**

接下来会调用`redis.c/demonize`设置为守护进程，接着调用**redis.c/initServer**为相关redisServer中结构分配空间。

`initServer`函数的主要流程是：

+ 设置信号处理函数，SIGTERM
+ 打开日志文件，openlog
+ 创建客户端连接链表、从服务器链表
+ 创建共享对象，如：常见回复ok,err,pong,-ERR no such key,1~10000直接的常用整数等
+ 调用aeCreateEventLoop创建事件循环结构体，重要
+ 初始化server.db数组，循环对每个数据库，即redisDb结构体，进行初始化
+ 打开监听端口
+ 注册监听事件，关联事件处理器accptTcpHandler，重要
+ 注册serverCron周期性事件，重要
+ 打开AOF或RDB文件
+ 初始化脚本系统
+ 初始化慢查询系统
+ 初始化BIO系统

initServer执行完毕后，会创建PID文件，设置服务器进程名字，下一步就应该从AOF或者RDB载入数据了。

**2.4 读取AOF或者RDB**

redis.c/main调用loadDataFromDisk函数，该函数内部根据AOF，还是RDB，调用loadAppendOnlyFile或者rdbLoad来载入文件。

AOF文件载入流程，即**loadAppendOnlyFile**函数流程：

1. 打开AOF文件
2. 创建伪客户端
3. 读取一行命令
4. 使用伪客户端执行命令
5. 文件是否读取完毕？否则继续执行3，是则下一步
6. 关闭文件，释放伪客户端

RDB文件载入流程，即**rdbLoad**函数流程大概为：打开文件，循环读取文件的databases部分（需要了解RDB文件的存储结构），直到EOF结束。

**2.5 主循环**

执行完上述初始化过程后，`redis.c/main`调用`aeMain`进行事件循环，去处理文件事件和周期性时间事件。

### 3 连接请求、命令处理、发送回复

**3.1 连接请求**

客户端的连接请求由`acceptTcpHandler`来处理，该函数接受连接、创建客户端、注册客户端的命令请求与回复处理器、添加到server.clients链表中。

监听套接字的请求处理器每次执行可以处理多个连接请求：

```
/*
 * 处理连接请求
 */
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
    char cip[REDIS_IP_STR_LEN];
    REDIS_NOTUSED(el);
    REDIS_NOTUSED(mask);
    REDIS_NOTUSED(privdata);
    //处理多个连接请求，至多MAX_ACCEPTS_PER_CALL，当连接没准备好时break（非阻塞）
    while(max--) {
        // accept 客户端连接
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        if (cfd == ANET_ERR) {
            if (errno != EWOULDBLOCK)
                redisLog(REDIS_WARNING,
                    "Accepting client connection: %s", server.neterr);
            return;
        }
        redisLog(REDIS_VERBOSE,"Accepted %s:%d", cip, cport);
        // 为客户端创建客户端状态（redisClient）、添加到server.clients链表，并注册读写事件处理器
        acceptCommonHandler(cfd,0);
    }
}
```

**3.2 命令请求**

先说下**命令总体的执行流程：**

1. 客户端键入命令，转换为Redis的协议格式，发送给服务器
2. 服务器的可写事件被激活，调用readQueryFromClient读入命令，存储到redisClient.queryBuf缓冲区中
3. 分析client缓冲区中的命令请求，查询命令表，保存到redisClient.cmd中
4. 调用命令的执行函数，redisCommand.proc
5. 做后续工作，比如慢查询日志、AOF工作
6. 向客户端返回回复

下面分步来说。

**客户端服务器通信协议**

Redis客户端与服务器的通信协议是如下格式：

```
*<参数数量>
$<参数 1 的字节数量> CR LF
<参数 1 的数据> CR LF
...
$<参数 N 的字节数量> CR LF
<参数 N 的数据> CR LF
```

举个栗子：

执行命令：`set mykey myvalue`

该命令的协议如下：

```
*3
$3
SET
$5
mykey
$7
myvalue</pre>
```

在redisClient.queryBuf保存为：`"*3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$7\r\nmyvalue\r\n"`

**readQueryFromClient读入命令**

```
/*
 * 读取客户端的查询缓冲区内容
 */
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    redisClient *c = (redisClient*) privdata;
    int nread, readlen;
    size_t qblen;
    // ...........省略
    // 分配空间，读入内容
    qblen = sdslen(c->querybuf);
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    nread = read(fd, c->querybuf+qblen, readlen);

    // 读入出错
    if (nread == -1) {
        if (errno == EAGAIN) { //非阻塞EAGIN
            nread = 0;
        } else {
            redisLog(REDIS_VERBOSE, "Reading from client: %s",strerror(errno));
            freeClient(c);
            return;
        }
    // read返回0，可能是客户端关闭了连接
    } else if (nread == 0) {
        redisLog(REDIS_VERBOSE, "Client closed connection");
        freeClient(c);
        return;
    }
    if (nread) {
        // 根据内容，更新查询缓冲区（SDS） free 和 len 属性，并将 '\0' 正确地放到内容的最后
        sdsIncrLen(c->querybuf,nread);
        // 记录服务器和客户端最后一次互动的时间
        c->lastinteraction = server.unixtime;
        // 如果客户端是 master 的话，更新它的复制偏移量
        if (c->flags & REDIS_MASTER) c->reploff += nread;
    } else {
        // 在 nread == -1 且 errno == EAGAIN 时运行
        server.current_client = NULL;
        return;
    }
    // ...省略部分
	
    // 从查询缓存重读取内容，创建参数，并执行命令
    // 函数会执行到缓存中的所有内容都被处理完为止
    processInputBuffer(c);

    server.current_client = NULL;
}
```

**processInputBuffer处理缓冲区**

其中会循环读取client.queryBuf，调用`processInlineBuffer`来解析命令，然后调用processCommand执行命令，重置客户端。

该过程可以如下伪代码表示：

```
void processInputBuffer(redisClient *c) {
    //循环读取缓冲区数据
    while(sdslen(c->querybuf)) {
	    //解析缓冲区命令
		processInlineBuffer(c);
		//执行命令
		processCommand(c);
		//重置客户端
		resetClient(c);
	}
}
```

<font color="ff0000">重点说一下redisCommand结构</font>

```
struct redisCommand {
    // 命令名字
    char *name;
    // 实现函数
    redisCommandProc *proc;
	//....
```

服务器通过`processInlineBuffer`解析命令后，保存到client.argc和client.argv中，然后调用`processCommand`，其内部会调用`lookupCommand`去查找命令表。

**命令表**是一个字典，保存在`redisServer.commands`中，字典的键是一个命令名字，比如set,get,del等；字典值是一个redisComand结构体，其记录了一个redis命令的实现信息。

查表的过程很简单，直接通过`dictFetchValue`即可，将查询结果保存到client.cmd中。

```
/*
 * 根据给定命令名字（SDS），查找命令
 */
struct redisCommand *lookupCommand(sds name) {
    return dictFetchValue(server.commands, name);
}
```

**执行命令**

`processCommand`中调用call函数，执行命令实现函数：

`call(c,REDIS_CALL_FULL); -->  c->cmd->proc(c);`

**call函数中更新统计信息**

**call函数**是真正执行命令的地方，是redis的核心函数，除了执行命令，还会统计命令执行时间，写入慢查询日志，发送给监视器，写入AOF，传播到slave节点。

```
// 调用命令的实现函数，执行命令
void call(redisClient *c, int flags) {
    //Note：其中省略了部分函数
	//转发给监视器
    replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
    // 计算命令开始执行的时间
    start = ustime();
    // 执行实现函数
    c->cmd->proc(c);
    // 计算命令执行耗费的时间
    duration = ustime()-start;
    // 计算命令执行之后的 dirty 值
    dirty = server.dirty-dirty;
    // 如果有需要，将命令放到 SLOWLOG 里面
    if (flags & REDIS_CALL_SLOWLOG && c->cmd->proc != execCommand)
        slowlogPushEntryIfNeeded(c->argv,c->argc,duration);
    // 更新命令的统计信息
    if (flags & REDIS_CALL_STATS) {
        c->cmd->microseconds += duration;
        c->cmd->calls++;
    }
    // 将命令复制到 AOF 和 slave 节点
    if (flags & REDIS_CALL_PROPAGATE) {
            propagate(c->cmd,c->db->id,c->argv,c->argc,flags);
    }
    server.stat_numcommands++;
}
```

**回复客户端**

在调用命令实现函数后，比如`setCommand`，它会调用`addReply`将回复保存到客户端的输出缓冲区中，并注册客户端套接字的可写事件，关联到sendReplyToClient事件处理器，当套接字可写时，会执行`networking.c/sendReplyToClient`，发送回复给客户端。

**经过上述步骤，就完成了Redis命令的执行过程。**

### 4 周期性事件serverCron内部流程

redis在初始化过程中注册了`serverCron`周期性事件，频率默认hz=10，即每100毫秒执行一次。

serverCron中完成众多工作，会定期对服务器自身的状态进行检查和调整，

1.  更新服务器的各类统计信息，比如时间、内存占用、每秒执行的命令次数
2.  调用databaseCron，清理数据库过期键值，对字典进行收缩操作
3.  处理SIGTERM信号，关闭服务器
4.  调用clientCron，关闭和清理失效客户端
5.  执行被延迟的BGRWRITEAOF，因为在BGSAVE期间，客户端的BGRWRITEAOF会被延迟
5.  检查BGSAVE和BGRWRITEAOF子进程的执行状态，如果已经执行完，则需要执行后续步骤
6.  将AOF缓冲区内容写入AOF文件
6.  如果是主服务器，则定义对从服务器同步
7.  集群模式，则定期同步和连接测试

**serverCron内部源码**

```
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    int j;

    if (server.watchdog_period) watchdogScheduleSignal(server.watchdog_period);

    /* Update the time cache. */
    updateCachedTime();

    // 记录服务器执行命令的次数
    run_with_period(100) trackOperationsPerSecond();

	//更新LRU时间
    server.lruclock = getLRUClock();

    // 记录服务器的内存峰值
    if (zmalloc_used_memory() > server.stat_peak_memory)
        server.stat_peak_memory = zmalloc_used_memory();

    // 服务器进程收到 SIGTERM 信号，会在sigtermHandler中打开shutdown_asap标志，在此处则关闭服务器
    if (server.shutdown_asap) {

        // 尝试关闭服务器
        if (prepareForShutdown(0) == REDIS_OK) exit(0);

        // 如果关闭失败，那么打印 LOG ，并移除关闭标识
        redisLog(REDIS_WARNING,"SIGTERM received but errors trying to shut down the server, check the logs for more information");
        server.shutdown_asap = 0;
    }

    // 打印数据库的键值对信息
    run_with_period(5000) {
        for (j = 0; j < server.dbnum; j++) {
            long long size, used, vkeys;

            // 可用键值对的数量
            size = dictSlots(server.db[j].dict);
            // 已用键值对的数量
            used = dictSize(server.db[j].dict);
            // 带有过期时间的键值对数量
            vkeys = dictSize(server.db[j].expires);

            // 用 LOG 打印数量
            if (used || vkeys) {
                redisLog(REDIS_VERBOSE,"DB %d: %lld keys (%lld volatile) in %lld slots HT.",j,used,vkeys,size);
                /* dictPrintStats(server.dict); */
            }
        }
    }

    // 如果服务器没有运行在 SENTINEL 模式下，那么打印客户端的连接信息
    if (!server.sentinel_mode) {
        run_with_period(5000) {
            redisLog(REDIS_VERBOSE,
                "%lu clients connected (%lu slaves), %zu bytes in use",
                listLength(server.clients)-listLength(server.slaves),
                listLength(server.slaves),
                zmalloc_used_memory());
        }
    }

    // 检查客户端，关闭超时客户端，并释放客户端多余的缓冲区
    clientsCron();

    // 对数据库执行各种操作，删除过期键，缩小字典
    databasesCron();

    // 如果 BGSAVE 和 BGREWRITEAOF 都没有在执行，并且有一个 BGREWRITEAOF 在等待，那么执行被延迟的BGREWRITEAOF
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
        server.aof_rewrite_scheduled)
    {
        rewriteAppendOnlyFileBackground();
    }

    // 检查 BGSAVE 或者 BGREWRITEAOF子进程是否已经执行完毕
    if (server.rdb_child_pid != -1 || server.aof_child_pid != -1) {
        int statloc;
        pid_t pid;

        // 接收子进程发来的信号，非阻塞
        if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {
            int exitcode = WEXITSTATUS(statloc);
            int bysignal = 0;

            if (WIFSIGNALED(statloc)) bysignal = WTERMSIG(statloc);

            // BGSAVE 执行完毕
            if (pid == server.rdb_child_pid) {
                backgroundSaveDoneHandler(exitcode,bysignal);

            // BGREWRITEAOF 执行完毕
            } else if (pid == server.aof_child_pid) {
                backgroundRewriteDoneHandler(exitcode,bysignal);

            } else {
                redisLog(REDIS_WARNING,
                    "Warning, detected child with unmatched pid: %ld",
                    (long)pid);
            }
            updateDictResizePolicy();
        }
    } else {

        // 既然没有 BGSAVE 或者 BGREWRITEAOF 在执行，那么检查是否需要执行它们
        // 即：判断是否满足变化参数，遍历所有保存条件，看是否需要执行 BGSAVE 命令
         for (j = 0; j < server.saveparamslen; j++) {
            struct saveparam *sp = server.saveparams+j;
            // 检查是否有某个保存条件已经满足了
            if (server.dirty >= sp->changes &&
                server.unixtime-server.lastsave > sp->seconds &&
                (server.unixtime-server.lastbgsave_try >
                 REDIS_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == REDIS_OK))
            {
                redisLog(REDIS_NOTICE,"%d changes in %d seconds. Saving...",
                    sp->changes, (int)sp->seconds);
                // 执行 BGSAVE
                rdbSaveBackground(server.rdb_filename);
                break;
            }
         }

        // 判断是否需要进行AOF重写，是则触发BGREWRITEAOF
         if (server.rdb_child_pid == -1 &&
             server.aof_child_pid == -1 &&
             server.aof_rewrite_perc &&
             // AOF 文件的当前大小大于执行 BGREWRITEAOF 所需的最小大小
             server.aof_current_size > server.aof_rewrite_min_size)
         {
            // 上一次完成 AOF 写入之后，AOF 文件的大小
            long long base = server.aof_rewrite_base_size ?
                            server.aof_rewrite_base_size : 1;

            // AOF 文件当前的体积相对于 base 的体积的百分比
            long long growth = (server.aof_current_size*100/base) - 100;

            // 如果增长体积的百分比超过了 growth ，那么执行 BGREWRITEAOF
            if (growth >= server.aof_rewrite_perc) {
                redisLog(REDIS_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
                // 执行 BGREWRITEAOF
                rewriteAppendOnlyFileBackground();
            }
         }
    }

    // 根据 AOF 政策，考虑是否需要将 AOF 缓冲区中的内容写入到 AOF 文件中
    run_with_period(1000) {
        if (server.aof_last_write_status == REDIS_ERR)
            flushAppendOnlyFile(0);
    }

    // 关闭那些需要异步关闭的客户端
    freeClientsInAsyncFreeQueue();

    // 清除被暂停的客户端
    clientsArePaused();

    // 主从复制的cron函数，周期性执行，默认1秒一次
    // 重连接主服务器、向主服务器发送 ACK 、判断数据发送失败情况、断开本服务器超时的从服务器，等等
    run_with_period(1000) replicationCron();

    // 集群的cron函数，周期性执行，默认1秒10次，向各个节点发送PING消息进行故障检测
    run_with_period(100) {
        if (server.cluster_enabled) clusterCron();
    }

    // sentinel 模式下的cron函数，周期性的发送INFO命令、PING命令、执行故障转移等
    run_with_period(100) {
        if (server.sentinel_mode) sentinelTimer();
    }

    // 集群操作相关，不懂此处
    run_with_period(1000) {
        migrateCloseTimedoutSockets();
    }

    // 增加 loop 计数器
    server.cronloops++;

    //返回时间间隔，代表周期性时间
    return 1000/server.hz;
}
```

### 5 Rerference

1.  黄健宏. Redis设计与实现[M]. 机械工业出版社, 2014.

* * *

**如果各位觉得这篇博客和代码对您有一定帮助，还请给[本博客](https://github.com/panzg123/panzg123.github.io)`star`一下，谢谢各位。**

作者：[panzg123](https://github.com/panzg123)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。