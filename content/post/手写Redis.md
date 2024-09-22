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
程序入口在redis.c中，从main函数可以看出,主要的操作在aeMain中，这就是redis中著名的ae库。
![[Pasted image 20240920194849.png]]
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
