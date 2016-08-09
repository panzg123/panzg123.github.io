---
layout: post
title: redis中主从复制的内部机制分析
categories: redis
---

主要介绍Redis中主从复制的机制，纯属个人理解。
====

### 1 背景

**主从复制**，是一项提升系统性能的常用手段，Redis也可以将多个实例配置成主从关系。

在Redis中，通过执行`SLAVEOF`命令或者slaveof选项，可以方便地配置主从关系。

Redis复制的特点：

+ 使用异步复制
+ 复用功能，不会阻塞主、从服务器

复制的原理？Redis2.8以前用`SYNC`命令来实现，内部原理是：

1. 主服务器执行BGSAVE，发送RDB，从服务器载入RDB
2. 命令传播，当主数据库被修改后，导致主从数据不一致，此时，主数据库将自己执行的命令也发送给从数据库，让从数据库也执行，这样来保证主从数据库的一致性。

关于PSYNC命令的原理将在后面介绍。

注：在 Redis 2.8 版本之前， 断线之后重连的从服务器总要执行一次完整重同步操作， 但是从 Redis 2.8 版本开始， 从服务器可以根据主服务器的情况来选择执行完整重同步还是部分重同步。

*   如果主服务器是 Redis 2.8 或以上版本，那么从服务器使用 **PSYNC** 命令来进行同步。
*   如果主服务器是 Redis 2.8 之前的版本，那么从服务器使用 **SYNC** 命令来进行同步。

下面将分别总结**完全同步、部分同步、命令传播、主从复制中的replicationCron、SLAVE of命令执行过程**。

### 2 SYNC完全同步

`SYNC命令`的处理过程为：

1. 从服务器向主服务器发送SYNC命令
2. 主服务器收到SYNC命令后，服务器开始执行BGSAVE命令，在后台生成一个RDB文件，并同时使用一个缓冲区记录开始执行后的所有命令
3. 主服务器的BGSAVE执行完毕后，将RDB文件发送给从服务器
4. 从服务器接收到RDB文件，载入RDB


**步骤1，从服务器发送SYNC命令**

1. 在``serverCron中，定期执行`replicationCron`，调用`connectWithMaster`，定期尝试连接主服务器。
2. 连接上后，注册连接套接字的读写事件，事件处理器为`syncWithMaster`，该处理器用于主从服务器定期同步。
3. 当syncWithMaster被触发时，会发送`PING\PONG\INFO`等消息，接着会注册一个读事件处理器`readSyncBulkPayload`，用来读取主服务器发送过来的RDB文件。
4. 当从数据库触发`readSyncBulkPayload`后，会创建临时rdb文件，接收完后，清空旧数据库，载入RDB文件，最后更新复制状态和偏移量。

注：第四步，要等主数据库发送RDB文件之后才能触发。

**步骤2，主服务器开始执行BGSAVE**

在`syncCommand`中，检查是否有`BGSAVE`正在执行？如果有，则判断是否有的别的slave也在等待这个RDB文件，如果有则表示可以复用这个RDB，如果没，只需要重新执行BGSAVE；如果没有BGSAVE在执行，也需要重新执行BGSAVE。

**步骤3，主服务器`serverCron`检测BGSAVE是否结束，并发送**

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

客户端发送**`PSYNC <runid> <offset>`**命令，主服务器收到该命令的处理流程为：

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

3. 如果进行部分同步，则`syncCommand`调用`addReplyReplicationBacklog`，发送挤压缓冲区中的数据。

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

### 5 replicationCron周期性函数

在复制机制中，有一个重要的周期性函数，`replicationCron`函数，其中会进行几项重要工作：

1. 尝试连接主服务器`connectWithMaster`（从服务器执行），在其中会注册主从服务器同步函数`syncWithMaster`，当执行`SLAVE OF`会激活该函数，该函数会执行一个完整的复制过程。

        // 尝试连接主服务器，这个事件会在repl_state==REDIS_REPL_CONNECT时被激活，而当执行slave of命令时，会把从服务器         // 置为这个状态
        if (server.repl_state == REDIS_REPL_CONNECT) {
            redisLog(REDIS_NOTICE,"Connecting to MASTER %s:%d",server.masterhost, server.masterport);
            if (connectWithMaster() == REDIS_OK) {
                redisLog(REDIS_NOTICE,"MASTER <-> SLAVE sync started");
            }
       }
        // SLAVE OF命令执行，将服务器设为指定地址的从服务器
        void replicationSetMaster(char *ip, int port) {
	          // 断开所有从服务器的连接，强制所有从服务器执行重同步
              disconnectSlaves(); /* Force our slaves to resync with us as well. */
              // 清空可能有的 master 缓存，因为已经不会执行 PSYNC 了
              replicationDiscardCachedMaster(); /* Don't try a PSYNC. */
              // 释放 backlog ，同理， PSYNC 目前已经不会执行了
              freeReplicationBacklog(); /* Don't allow our chained slaves to PSYNC. */
              // 取消之前的复制进程（如果有的话）
              cancelReplicationHandshake();
              // 进入连接状态（重点）
              server.repl_state = REDIS_REPL_CONNECT;
              server.master_repl_offset = 0;
	     }

2. 定期向主服务器发送`REPLCONF ACK <replication_offset>`,告知当前复制偏移量。

        if (server.masterhost && server.master && !(server.master->flags & REDIS_PRE_PSYNC))
	        //发送ACK命令
            replicationSendAck();

3. 如果服务器有从服务器，定时向它们发送 PING 。

        ping_argv[0] = createStringObject("PING",4);
        replicationFeedSlaves(server.slaves, server.slaveseldb, ping_argv, 1);

4. 断开超时的从服务器，`repl_ack_time`时间会在`replCommand`中更新

        // 遍历所有从服务器
        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            redisClient *slave = ln->value;
            ...
            // 释放超时从服务器，即repl_ack时间过久
            if ((server.unixtime - slave->repl_ack_time) > server.repl_timeout)
            {
                char ip[REDIS_IP_STR_LEN];
                int port;

                if (anetPeerToString(slave->fd,ip,sizeof(ip),&port) != -1) {
                    redisLog(REDIS_WARNING,
                        "Disconnecting timedout slave: %s:%d",
                        ip, slave->slave_listening_port);
                }
                // 释放
                freeClient(slave);
            }
        }

### 6 syncWithMaster一个完整的同步过程

上面讲，当从服务器执行`SLAVE OF`，结合`replicationCron`，就会注册``syncWithMaster``事件处理器。其中就是一个完整的同步过程，用到了`SYNC`和`PSYNC`同步。

该函数主要执行以下过程：

1. 发送`PING`消息

         if (server.repl_state == REDIS_REPL_CONNECTING) {
		     // 更新状态
             server.repl_state = REDIS_REPL_RECEIVE_PONG;
			 // 同步发送 PING
             syncWrite(fd,"PING\r\n",6,100);
		 }
2. 接收`PONG`

        // 接收 PONG 命令
        if (server.repl_state == REDIS_REPL_RECEIVE_PONG) {
		    // 同步接收 PONG
            if (syncReadLine(fd,buf,sizeof(buf),server.repl_syncio_timeout*1000) == -1)
            {
                redisLog(REDIS_WARNING,"I/O error reading PING reply from master: %s",strerror(errno));
                goto error;
            }
		}
3. 进行身份验证

        // 进行身份验证
        if(server.masterauth) {
            err = sendSynchronousCommand(fd,"AUTH",server.masterauth,NULL);
            sdsfree(err);
        }

4. 从服务器发送端口信息，即发送`REPLCONF listening-port <port-number>`，主服务器收到该消息后，会更新自身关于从服务器的记录。

        sds port = sdsfromlonglong(server.port);
        err = sendSynchronousCommand(fd,"REPLCONF","listening-port",port,NULL);

5. 调用slaveTryPartialResynchronization，通过发送`PSYNC`命令，根据返回结果，判断是进行全同步还是部分同步。

        // 根据返回的结果决定是执行部分 resync ，还是 full-resync
        psync_result = slaveTryPartialResynchronization(fd);

6. 若不能执行部分同步，则发送`SYNC`命令，通知主服务器全同步，发送`RDB`文件。

        // 主服务器不支持 PSYNC ，发送 SYNC
        if (psync_result == PSYNC_NOT_SUPPORTED) {
		    syncWrite(fd,"SYNC\r\n",6,server.repl_syncio_timeout*1000)；
		}

7. 从服务器打开临时文件，注册套接字的读事件，关联`readSyncBulkPayload`事件处理器，准备接收RDB文件。

        // 打开临时文件，准备接收RDB
        snprintf(tmpfile,256,"temp-%d.%ld.rdb",(int)server.unixtime,(long int)getpid());
        dfd = open(tmpfile,O_CREAT|O_WRONLY|O_EXCL,0644);
        // 设置一个读事件处理器，来读取主服务器的 RDB 文件
        if (aeCreateFileEvent(server.el,fd, AE_READABLE,readSyncBulkPayload,NULL)
		{
		}
		// 设置状态
        server.repl_state = REDIS_REPL_TRANSFER;
		server.repl_transfer_fd = dfd;
		server.repl_transfer_tmpfile = zstrdup(tmpfile);

###  Rerference

1.  黄健宏. Redis设计与实现[M]. 机械工业出版社, 2014.


* * *

**如果各位觉得这篇博客和代码对您有一定帮助，还请给[本博客](https://github.com/panzhengguang/panzhengguang.github.io)`star`一下，谢谢各位。**

作者：[panzhengguang](https://github.com/panzhengguang)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。