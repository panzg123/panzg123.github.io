---
layout: post
title: 定时器编程小记
categories: 网络编程
---
## 一、定时器

计算机系统及程序通常需要处理一些定时事件，比如：比如tcp协议中利用定时器管理包超时，视频显示中利用定时器来定时显示视频帧，web服务中利用定时器来管理用户的活动状态。

通常在服务器编程中需要处理多种定时事件，因此有效的组织这些定时事件，至关重要。

此时，我们需要将每个定时事件分别封装成定时器，并使用某种容器数据结构，比如链表、最小堆等方式，来将所有定时器串联起来，以实现对定时事件的统一管理。

## 二、定时器的实现

总之，定时就是指在一段时间后触发某段代码的机制。

Linux通常提供三种定时方法：

*   socket选项SO_RECVTIMEO和SO_SENDTIMEO
*   SIGALRM信号
*   I/O复用系统调用的超时参数

### 2.1 socket选项

socekt选项SO_RECVTIMEO和SO_SENDTIMEO，分别用来设置socket接受数据超时和发送数据超时。

用于socket专用的系统调用：send,sendmsg,recv,recvmsg,accept,connect等。

demo：  

```  
    /*数据结构 timeval*/
    struct timeval timeout;
    timeout.tv_sec = time;
    timeout.tv_usec = 0;
    socklen_t len = sizeof( timeout ); /*设置sockfd超时参数*/ ret = setsockopt( sockfd, SOL_SOCKET, SO_SNDTIMEO, &timeout, len );
    assert( ret != -1 );
    ret = connect( sockfd, ( struct sockaddr* )&address, sizeof( address ) ); if ( ret == -1 )
    { /*连接超时*/
        if( errno == EINPROGRESS )
        {
            printf( "connecting timeout\n" ); return -1;
        }
        printf( "error occur when connecting to server\n" ); return -1;
    }
```  

### 2.2  SIGALRM信号

主要使用alarm 和 setitimer系统调用来设置实时闹钟，触发SIGALRM信号，利用该信号来处理定时任务。如果处理多个定时任务，则需要不断地触发SIGALRM信号。

下面实现一种简单的定时器，基于升序链表的定时器，并用来监测非活动连接。

主要思路：定时器至少包含两个成员：超时时间和任务回调函数，使用链表进行连接。

源码说明：lst_timer.h是定时器类，nonactive_conn.cpp是用户监测非活动连接的主函数。


``` 
/*
 * lst_timer.h
 *简单升序定时器链表
 *  Created on: 2015-4-30
 *      Author: panzg
 */

#ifndef LST_TIMER_H_
#define LST_TIMER_H_

#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <signal.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <pthread.h>
#include <time.h>

#define BUFFER_SIZE 64
class util_timer;
//用户数据结构
struct client_data
{
    sockaddr_in address;
    int sockfd;
    char buf[BUFFER_SIZE];
    util_timer* timer;
};

//定时器类
class util_timer
{
public:
    util_timer():prev(NULL),next(NULL){}

public:
    time_t expire;  /*任务的超时时间，这里使用了绝对时间*/
    void (*cb_func)(client_data*); /*任务的回调函数*/
    client_data*user_data;
    util_timer* prev;
    util_timer* next;
};

class sort_timer_lst
{
private:
    util_timer* head;
    util_timer* tail;
public:
    sort_timer_lst():head(NULL),tail(NULL)  { }
    ~sort_timer_lst()
    {
        util_timer * tmp = head;
        while(tmp)
        {
            head = tmp->next;
            delete tmp;
            tmp = head;
        }
    }
    /*add_timer*/
    void add_timer(util_timer* timer)
    {
        if(!timer)   //timer为空
        {
            return;
        }
        if(!head)  //头节点为空
        {
            head=tail=timer;
            return;
        }
        if(timer->expire<head->expire) //时间小于head->expire
        {
            timer->next = head;
            head->prev = timer;
            head = timer;
            return;
        }
        //否则，插入链表合适位置
        add_timer(head,timer);
    }
    /*定时器任务发生变化，则需要调整在链表中的合适位置。只考虑定时器的
     * 超时时间延长的情况*/
    void adjust_timer( util_timer* timer )
        {
            if( !timer )
            {
                return;
            }
            util_timer* tmp = timer->next;
            /*若调整时间后，仍然小于下一个节点的时间，则不许调整*/
            if( !tmp || ( timer->expire < tmp->expire ) )
            {
                return;
            }
            /*若该节点是链表头节点*/
            if( timer == head )
            {
                head = head->next;
                head->prev = NULL;
                timer->next = NULL;
                add_timer( timer, head );
            }
            /*若不是头节点，删除重新插入*/
            else
            {
                timer->prev->next = timer->next;
                timer->next->prev = timer->prev;
                add_timer( timer, timer->next );
            }
        }
    /*目标定时器timer从链表中删除*/
    void del_timer( util_timer* timer )
    {
        if( !timer )
        {
            return;
        }
        //只有一个节点
        if( ( timer == head ) && ( timer == tail ) )
        {
            delete timer;
            head = NULL;
            tail = NULL;
            return;
        }
        //两个以上节点，但是 timer == head
        if( timer == head )
        {
            head = head->next;
            head->prev = NULL;
            delete timer;
            return;
        }
        //两个以上节点，但是 timer == tail
        if( timer == tail )
        {
            tail = tail->prev;
            tail->next = NULL;
            delete timer;
            return;
        }
        //两个以上节点，目标节点位置中间
        timer->prev->next = timer->next;
        timer->next->prev = timer->prev;
        delete timer;
    }
    /*SIGALRM信号的每次被触发，都要在其信号处理函数中执行一次tick函数，以处理
     * 链表上的到期任务*/
    void tick()
       {
           if( !head )
           {
               return;
           }
           printf( "timer tick\n" );
           //获取当前的系统时间
           time_t cur = time( NULL );
           util_timer* tmp = head;
           //从头节点开始处理，直到 未到期的定时器
           while( tmp )
           {
               if( cur < tmp->expire )
               {
                   break;
                }
               //调用定时器的回调函数，以执行定时任务
               tmp->cb_func( tmp->user_data );
               //执行后，从链表中删除，并重置
               head = tmp->next;
               if( head )
               {
                   head->prev = NULL;
               }
               delete tmp;
               tmp = head;
           }
       }

private:
    /*重载的函数，被add_timer和adjust_timer调用，作用是将目标定时器添加到
     * 节点lst_head之后的合适位置*/
        void add_timer( util_timer* timer, util_timer* lst_head )
        {
            util_timer* prev = lst_head;
            util_timer* tmp = prev->next;
            //查找合适位置
            while( tmp )
            {
                if( timer->expire < tmp->expire )
                {
                    prev->next = timer;
                    timer->next = tmp;
                    tmp->prev = timer;
                    timer->prev = prev;
                    break;
                }
                prev = tmp;
                tmp = tmp->next;
            }
            //在尾部
            if( !tmp )
            {
                prev->next = timer;
                timer->prev = prev;
                timer->next = NULL;
                tail = timer;
            }

        }
};

#endif /* LST_TIMER_H_ */
``` 

**nonactive_conn.cpp**

``` 
/* * nonactive_conn.cpp
 *处理非活动连接
 *利用alarm函数周期性触发SIGALRM信号，执行定时器的任务----关闭非活动的连接
 *  Created on: 2015-4-30
 *      Author: panzg */
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <signal.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <pthread.h>
#include "lst_timer.h"

#define FD_LIMIT 65535
#define MAX_EVENT_NUMBER 1024
#define TIMESLOT 5

static int pipefd[2];
static sort_timer_lst timer_lst;
static int epollfd = 0;

int setnonblocking( int fd )
{
    int old_option = fcntl( fd, F_GETFL );
    int new_option = old_option | O_NONBLOCK;
    fcntl( fd, F_SETFL, new_option );
    return old_option;
}

void addfd( int epollfd, int fd )
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN | EPOLLET;
    epoll_ctl( epollfd, EPOLL_CTL_ADD, fd, &event );
    setnonblocking( fd );
}

void sig_handler( int sig )
{
    int save_errno = errno;
    int msg = sig;
    send( pipefd[1], ( char* )&msg, 1, 0 );
    errno = save_errno;
}

void addsig( int sig )
{
    struct sigaction sa;
    memset( &sa, '\0', sizeof( sa ) );
    sa.sa_handler = sig_handler;
    sa.sa_flags |= SA_RESTART;
    sigfillset( &sa.sa_mask );
    assert( sigaction( sig, &sa, NULL ) != -1 );
}

void timer_handler()
{
    timer_lst.tick();
    alarm( TIMESLOT );
}

/*定时器回调函数，删除非活动连接socket上的注册事件，并关闭*/
void cb_func(client_data* user_data)
{
    epoll_ctl(epollfd,EPOLL_CTL_DEL,user_data->sockfd,0);
    assert(user_data);
    close(user_data->sockfd);
    printf("close fd %d\n",user_data->sockfd);
}

int main(int argc, char* argv[])
{
        if( argc <= 2 )
        {
            printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
            return 1;
        }
        const char* ip = argv[1];
        int port = atoi( argv[2] );

        int ret = 0;
        struct sockaddr_in address;
        bzero( &address, sizeof( address ) );
        address.sin_family = AF_INET;
        inet_pton( AF_INET, ip, &address.sin_addr );
        address.sin_port = htons( port );

        int listenfd = socket( PF_INET, SOCK_STREAM, 0 );
        assert( listenfd >= 0 );

        ret = bind( listenfd, ( struct sockaddr* )&address, sizeof( address ) );
        assert( ret != -1 );

        ret = listen( listenfd, 5 );
        assert( ret != -1 );

        epoll_event events[ MAX_EVENT_NUMBER ];
         int epollfd = epoll_create( 5 );
         assert( epollfd != -1 );
         addfd( epollfd, listenfd );

         ret = socketpair( PF_UNIX, SOCK_STREAM, 0, pipefd );
         assert( ret != -1 );
         setnonblocking( pipefd[1] );
         addfd( epollfd, pipefd[0] );

         /*设置信号处理函数*/
         addsig(SIGALRM);
         addsig(SIGTERM);
         bool stop_server = false;

         client_data* users = new client_data[FD_LIMIT];
         bool timeout = false;
         alarm( TIMESLOT );

         while(!stop_server)
         {
             int number = epoll_wait( epollfd, events, MAX_EVENT_NUMBER, -1 );
             if ( ( number < 0 ) && ( errno != EINTR ) )
               {
               printf( "epoll failure\n" );
               break;
            }
             for ( int i = 0; i < number; i++ )
                   {
                       int sockfd = events[i].data.fd;
                       /*新客户的到来，登记*/
                       if( sockfd == listenfd )
                       {
                           struct sockaddr_in client_address;
                           socklen_t client_addrlength = sizeof( client_address );
                           int connfd = accept( listenfd, ( struct sockaddr* )&client_address, &client_addrlength );
                           addfd( epollfd, connfd );
                           users[connfd].address = client_address;
                           users[connfd].sockfd = connfd;
                           util_timer* timer = new util_timer;
                           timer->user_data = &users[connfd];
                           timer->cb_func = cb_func;
                           time_t cur = time( NULL );
                           timer->expire = cur + 3 * TIMESLOT;
                           users[connfd].timer = timer;
                           timer_lst.add_timer( timer );
                       }
                       /*处理信号*/
                       else if( ( sockfd == pipefd[0] ) && ( events[i].events & EPOLLIN ) )
                       {
                           int sig;
                           char signals[1024];
                           ret = recv( pipefd[0], signals, sizeof( signals ), 0 );
                           if( ret == -1 )
                           {
                               // handle the error
                               continue;
                           }
                           else if( ret == 0 )
                           {
                               continue;
                           }
                           else
                           {
                               for( int i = 0; i < ret; ++i )
                               {
                                   switch( signals[i] )
                                   {
                                       case SIGALRM:
                                       {
                                           timeout = true;
                                           break;
                                       }
                                       case SIGTERM:
                                       {
                                           stop_server = true;
                                       }
                                   }
                               }
                           }
                       }
                       //接受来自客户端的数据，并且调整定时器
                       else if(  events[i].events & EPOLLIN )
                       {
                           memset( users[sockfd].buf, '\0', BUFFER_SIZE );
                           ret = recv( sockfd, users[sockfd].buf, BUFFER_SIZE-1, 0 );
                           printf( "get %d bytes of client data %s from %d\n", ret, users[sockfd].buf, sockfd );
                           util_timer* timer = users[sockfd].timer;
                           if( ret < 0 )
                           {
                               /*发生读错误，关闭连接，移除定时器*/
                               if( errno != EAGAIN )
                               {
                                   cb_func( &users[sockfd] );
                                   if( timer )
                                   {
                                       timer_lst.del_timer( timer );
                                   }
                               }
                           }
                           else if( ret == 0 )
                           {
                               /*对方关闭了连接，此时需要关闭连接，移除定时器*/
                               cb_func( &users[sockfd] );
                               if( timer )
                               {
                                   timer_lst.del_timer( timer );
                               }
                           }
                           else
                           {
                               /*如果客户端上有数据可以读，则需要调整定时器*/
                               //send( sockfd, users[sockfd].buf, BUFFER_SIZE-1, 0 );
                               if( timer )
                               {
                                   time_t cur = time( NULL );
                                   timer->expire = cur + 3 * TIMESLOT;
                                   printf( "adjust timer once\n" );
                                   timer_lst.adjust_timer( timer );
                               }
                           }
                       }
                       else
                       {
                           // others
                       }
                   }

             /*最后处理定时事件，IO事件具有更高的优先级，当然也导致定时任务不能
              * 精确时间执行*/
                   if( timeout )
                   {
                       timer_handler();
                       timeout = false;
                   }
         }
}
``` 

此外，还可以采用hash表和最小堆来组织定时器。

### 2.3 I/O复用系统调用的超时参数

Linux下的三种I/O复用系统都带有超时参数，它们不仅能统一处理新号和I/O事件，也能同意处理定时事件。

但由于I/O复用系统调用可能在超时时间到期之前就返回，所以如果我们要利用它们来定时，就需要不断更新定时参数以反映剩余的时间。

demo：

```
#define TIMEOUT 5000

int timeout = TIMEOUT;
time_t start = time( NULL );
time_t end = time( NULL );
while( 1 )
{
    printf( "the timeout is now %d mill-seconds\n", timeout );
    start = time( NULL );
    int number = epoll_wait( epollfd, events, MAX_EVENT_NUMBER, timeout );
    if( ( number < 0 ) && ( errno != EINTR ) )
    {
        printf( "epoll failure\n" );
        break;
    }
    /*epoll_wait返回0，说明超时时间到，此时便可以处理定时任务，并重置定时时间*/
    if( number == 0 )
    {
        timeout = TIMEOUT;
        continue;
    }
    /*更新timeout，获得下次epoll_wait调用的超时参数*/
    end = time( NULL );
    timeout -= ( end - start ) * 1000;
    if( timeout <= 0 )
    {
        timeout = TIMEOUT;
    }

    // handle connections
}
```

作者：[西芒xiaoP](http://www.cnblogs.com/panweishadow/)

出处：迁移自我的博客园 [http://www.cnblogs.com/panweishadow/](http://www.cnblogs.com/panweishadow/) 

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。