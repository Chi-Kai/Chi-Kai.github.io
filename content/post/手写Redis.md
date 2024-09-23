---
author: KC
type: post
tags:
  - Redis
date: 2024-07-20
description: 源码学习
title: "手写Redis"
toc: true
draft: false
---
# Redis 架构解析 (Redis 1.0)
程序入口在redis.c中，从main函数可以看出,主要的操作在aeMain中，这就是redis中著名的ae库。下面我们先介绍一下ae库:
![[Pasted image 20240920194849.png]]
### ae库
aeMain作用是持续处理aeloop中的事务:
![[Pasted image 20240920195230.png]]
ae库主要关注的是三个结构体，定义了FileEvent和TimeEvent，以及事件循环
```c
/* File event structure */

typedef struct aeFileEvent {

    int fd;

    int mask; /* one of AE_(READABLE|WRITABLE|EXCEPTION) */

    aeFileProc *fileProc; // 回调函数

    aeEventFinalizerProc *finalizerProc; // 事件完成时回调

    void *clientData;

    struct aeFileEvent *next;

} aeFileEvent;

  

/* Time event structure */

typedef struct aeTimeEvent {

    long long id; /* time event identifier. */

    long when_sec; /* seconds */

    long when_ms; /* milliseconds */

    aeTimeProc *timeProc;

    aeEventFinalizerProc *finalizerProc;

    void *clientData;

    struct aeTimeEvent *next;

} aeTimeEvent;

  

/* State of an event based program */

typedef struct aeEventLoop {

    long long timeEventNextId;

    aeFileEvent *fileEventHead;

    aeTimeEvent *timeEventHead;

    int stop;

} aeEventLoop;
```
ae库最主要的函数就是`aeProcessEvents` , 它的主要功能是处理文件事件（如网络套接字读写事件）和时间事件（如定时器事件）。它会根据传入的标志位（`flags`）决定是否处理这两种事件，并返回处理的事件数量。
下面讲一下大概的流程:
```c
    /* Check file events */
    if (flags & AE_FILE_EVENTS) {
        while (fe != NULL) {
            if (fe->mask & AE_READABLE) FD_SET(fe->fd, &rfds);
            if (fe->mask & AE_WRITABLE) FD_SET(fe->fd, &wfds);
            if (fe->mask & AE_EXCEPTION) FD_SET(fe->fd, &efds);
            if (maxfd < fe->fd) maxfd = fe->fd;
            numfd++;
            fe = fe->next;
        }
    }
```
如果 `flags` 中设置了 `AE_FILE_EVENTS`，则遍历文件事件链表，将需要监听的文件描述符添加到 `rfds`, `wfds`, `efds` 中，并更新 `maxfd` 和 `numfd`。
`rfds wfds efds` 是类型为`fd_set`的描述符集合。
然后它做了哪些事情呢:
1. 获取当前到最新时间事件的差，作为select的超时时间。
2. 处理 `select` 返回的事件。
	- 如果 `select` 返回值大于0，表示有事件发生，遍历文件事件链表，检查哪些文件描述符上有事件发生，并调用相应的处理函数 `fileProc`。
	- 处理完事件后，重新从链表头开始遍历，因为事件处理可能会修改链表结构。
3. 处理时间事件：
	- 如果 flags 中设置了 AE_TIME_EVENTS，则遍历时间事件链表，检查哪些时间事件已经到期，并调用相应的处理函数 timeProc。
	- 如果时间事件需要重复触发，则更新其触发时间；否则，删除该时间事件。
4. 返回处理的事件数量。
我们接着看回main函数,在事件循环中注册了一个server的acceptHandler函数，一旦有客户端链接可读，就触发。
![[Pasted image 20240922161721.png]]
![[Pasted image 20240922164350.png]]
其中这个函数有两个操作 accept客户端请求和创建一个新的客户端。我们看这个createClient函数,除了设置一些客户端状态外，还为每个客户端注册了一个处理请求的函数readQueryFromClient。当客户端fd可读就触发
![[Pasted image 20240922164641.png]]
### 一条命令的执行
`readQueryFromClient` 函数的主要作用是从客户端读取数据，解析查询请求，并将其传递给 Redis 服务器进行处理。
1. 读数据，将读到的数据写入缓冲区。这里可以看出，这个操作是持续进行的，也就是客户端可以保持这个链接，一旦客户端fd可读，就写入buffer。可以连续操作
![[Pasted image 20240923094649.png]]
2. 处理数据。首先将收到的字符串分离处理，把命令和参数分开。这里就不细看了。处理完之后，命令被放到argv中，参数放到argc中。可以看到这里有一个跳转语句goto。当buffer还有命令时会重复执行。这里还有一个bulklen参数，是判断处理批量读取操作。
```c
again:
	if (c->bulklen == -1) {
    char *p = strchr(c->querybuf, '\n');
    if (p) {
        // 解析命令和参数
    } else if (sdslen(c->querybuf) >= REDIS_REQUEST_MAX_SIZE) {
        redisLog(REDIS_DEBUG, "Client protocol error");
        freeClient(c);
        return;
	}
  }
// 执行语句
if (c->argc && processCommand(c) && sdslen(c->querybuf)) goto again;
```
下面看一条命令是怎么执行的。
