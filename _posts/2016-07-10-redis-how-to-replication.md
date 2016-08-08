---
layout: post
title: redis中主从复制的内部机制分析
categories: redis
---

主要介绍Redis中主从复制的机制，纯属个人理解。
====

### 1 背景

**主从复制**，是一项提升系统性能的常用手段，Redis也可以将多个实例配置成主从关系。

在Redis中，通过执行SLAVEOF命令或者slaveof选项，可以方便地配置主从关系。

Redis复制的特点：

+ 使用异步复制
+ 复用功能，不会阻塞主、从服务器

复制的原理？Redis2.8以前用SYNC命令来实现，内部原理是：

1. 主服务器执行BGSAVE，发送RDB，从服务器载入RDB
2. 命令传播，当主数据库被修改后，导致主从数据不一致，此时，主数据库将自己执行的命令也发送给从数据库，让从数据库也执行，这样来保证主从数据库的一致性。

关于PSYNC命令的原理将在后面介绍。

注：在 Redis 2.8 版本之前， 断线之后重连的从服务器总要执行一次完整重同步（full resynchronization）操作， 但是从 Redis 2.8 版本开始， 从服务器可以根据主服务器的情况来选择执行完整重同步还是部分重同步（partial resynchronization）。

*   如果主服务器是 Redis 2.8 或以上版本，那么从服务器使用 **PSYNC** 命令来进行同步。
*   如果主服务器是 Redis 2.8 之前的版本，那么从服务器使用 **SYNC** 命令来进行同步。

下面将分别总结**完全同步、部分同步、命令传播**。

### 2 SYNC完全同步

完全同步的执行步骤为：

1. 从服务器向主服务器发送SYNC命令
2. 收到SYNC命令后，服务器开始执行BGSAVE命令，在后台生成一个RDB文件，并同时使用一个缓冲区记录开始执行后的所有命令
3. 主服务器的BGSAVE执行完毕后，将RDB文件发送给从服务器
4. 从服务器接收到RDB文件，载入RDB


**步骤1，发送SYNC命令**

1. 在``serverCron中，定期执行`replicationCron`，调用`connectWithMaster`，定期尝试连接主服务器。
2. 连接上后，注册连接套接字的读写事件，事件处理器为`syncWithMaster`，该处理器用于主从服务器定期同步。
3. 当syncWithMaster被触发时，会发送`PING\PONG\INFO`等消息，接着会注册一个读事件处理器`readSyncBulkPayload`，用来读取主服务器发送过来的RDB文件。
4. 当从数据库触发`readSyncBulkPayload`后，会创建临时rdb文件，接收完后，清空旧数据库，载入RDB文件，最后更新复制状态和偏移量。

注：第四步，要等主数据库发送RDB文件之后才能触发。

**步骤2，开始执行BGSAVE**

在`syncCommand`中，检查是否有`BGSAVE`正在执行？如果有，则判断是否有的别的slave也在等待这个RDB文件，如果有则表示可以复用这个RDB，如果没，只需要重新执行BGSAVE；如果没有BGSAVE在执行，也需要重新执行BGSAVE。

**步骤3，serverCron检测BGSAVE是否结束，并发送**

主服务器的`serverCron`会定期检查子进程RDB是否完成，完成后，调用`backgroundSaveDoneHandler`，其中再调用`updateSlavesWaitingBgsave`来处理那些等待BGSAVE完成的slave，在`updateSlavesWaitingBgsave`中会注册套接字的可写事件，当可写时，触发`sendBulkToSlave`发送RDB文件给slave。

```
if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {
    if (pid == server.bgsavechildpid) {
        backgroundSaveDoneHandler(statloc);
    } 
```

函数调用层次关系：

`serverCron -->   backgroundSaveDoneHandler  -->  updateSlavesWaitingBgsave  -->  注册sendBulkToSlave`

通过调用sendBulkToSlave，可以将RDB文件循环发送给slave。

**步骤4，从服务器接收RDB，载入**

在上面的步骤1中，触发`readSyncBulkPayload`会接收RDB并载入。

### 3 PSYNC部分同步

部分重同步的功能，有3部分组成：

1. 主服务器的复制偏移量和从服务器的复制偏移量
2. 主服务器的复制积压缓冲区，缓冲区是一个固定大小的先进先出的列队
3. 服务器的运行ID

是否适用部分同步的检查办法：

*   如果从服务器记录的主服务器 ID 和当前要连接的主服务器的 ID 相同，并且从服务器记录的偏移量所指定的数据仍然保存在主服务器的复制流缓冲区里面，那么主服务器会向从服务器发送断线时缺失的那部分数据，然后复制工作可以继续执行。
*   否则的话，从服务器就要执行完整重同步操作。

客户端发送**`PSYNC <runid> <offset>`**命令后，处理流程：

1. 主服务器调用`replication.c/syncCommand`函数，其中首先会尝试调用`masterTryPartialResynchronization`部分同步。
   
       if (!strcasecmp(c->argv[0]->ptr,"psync")) {
            // 尝试进行 PSYNC
            if (masterTryPartialResynchronization(c) == REDIS_OK) {
		    }
	   }

2. 在`masterTryPartialResynchronization`中检查`runid`和`offset`，如果需要全同步，向客户端返回`FULLRESYNC`回复；如果满足要求，返回`CONTINUE`回复。
        
		if (strcasecmp(master_runid, server.runid)) {
		    goto need_full_resync; //需要 full resync
		}
		if (psync_offset参数不符合要求){
		    goto need_full_resync; //需要 full resync
		}

3. 如果进行部分同步，则调用`addReplyReplicationBacklog`，发送挤压缓冲区中的数据。

        // 发送 backlog 中的内容（也即是从服务器缺失的那些内容）到从服务器
        psync_len = addReplyReplicationBacklog(c,psync_offset);

4. 如果第二步中，`goto need_full_resync`，则返回`REDIS_ERR`，到`syncCommand`中继续执行后面的全同步过程，过程就和`SYNC`过程一致。

注：从服务器的第一次复制，发送的是`PSYNC ? -1`，会被主服务器强制完全同步。

### 4 命令传播

经过上面的同步之后，主从数据库一致了，此后服务器进入命令传播阶段，主服务器一直把自己执行的写命令发送给服务器，从服务器一致接收并执行主服务器发送过来的命令，就可以保证主从服务器一致了。

该过程提现在call函数中，如果在call函数执行过程中发现数据的修改，就会进行传播。具体函数调用为：

`call()->propagate()->replicationFeedSlaves()`

1. `call`函数中判断是否修改？
    
	    void call(redisClient *c, int flags) {
		     // 如果数据库有被修改，那么启用 REPL 和 AOF 传播
            if (dirty)
                flags |= (REDIS_PROPAGATE_REPL | REDIS_PROPAGATE_AOF);
			if (flags != REDIS_PROPAGATE_NONE)
			    //传播命令
                propagate(c->cmd,c->db->id,c->argv,c->argc,flags);
		}
		
2. 在`propagate`函数中调用`replicationFeedSlaves`，传播到slave端

        void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc,int flags)
        {
            // 传播到 slave
            if (flags & REDIS_PROPAGATE_REPL)
                replicationFeedSlaves(server.slaves,dbid,argv,argc);
        }
		
3. 在`replicationFeedSlaves`中，循环发送数据给所有的slave，同时把数据放到back_log积压空间中

        void replicationFeedSlaves(list *slaves, int dictid, robj **argv, int argc) {
		    // 发送给所有从服务器
			listRewind(slaves,&li);
            while((ln = listNext(&li))) {
                redisClient *slave = ln->value;
                addReplyMultiBulkLen(slave,argc);
                for (j = 0; j < argc; j++)
                    addReplyBulk(slave,argv[j]);
               }
		}

4. 从服务器通过相应的事件处理器接收命令，并执行，保证一致性。

### 5 心跳检测

在命令传播阶段，从服务器会默认以每秒一次的频率，向主服务器发送命令：`REPLCONF ACK <replication_offset>`，其主要作用有是确认连接正常，告知当前复制偏移量。

```
void replicationCron(void) {
    ...
    if (server.masterhost && server.master && !(server.master->flags & REDIS_PRE_PSYNC))
	    //发送ACK命令
        replicationSendAck();
}

void replconfCommand(redisClient *c) {
     .....
	 if (!strcasecmp(c->argv[j]->ptr,"ack")) {
            // 从服务器使用 REPLCONF ACK 告知主服务器，
            // 从服务器目前已处理的复制流的偏移量
            // 主服务器更新它的记录值
            // 也即是 INFO replication 中的  slaveN ..., offset = xxx 这一项
            long long offset;

            if (!(c->flags & REDIS_SLAVE)) return;
            if ((getLongLongFromObject(c->argv[j+1], &offset) != REDIS_OK))
                return;
            // 如果 offset 已改变，那么更新
            if (offset > c->repl_ack_off)
                c->repl_ack_off = offset;
            // 更新最后一次发送 ack 的时间
            c->repl_ack_time = server.unixtime;
            /* Note: this command does not reply anything! */
            return;
        }
        .......
}
```

### 6 Rerference

1.  黄健宏. Redis设计与实现[M]. 机械工业出版社, 2014.

* * *

**如果各位觉得这篇博客和代码对您有一定帮助，还请给[本博客](https://github.com/panzhengguang/panzhengguang.github.io)`star`一下，谢谢各位。**

作者：[panzhengguang](https://github.com/panzhengguang)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。