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
程序入口在redis.c中，从main函数可以看出,主要的操作在aeMain中
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
