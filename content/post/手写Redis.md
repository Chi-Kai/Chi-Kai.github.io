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
程序入口在redis.c中，从main函数可以看出,主要的操作的aeMain中
![[Pasted image 20240920194849.png]]
aeMain在ae.c中。可以看出aeMain就是持续处理事务:
![[Pasted image 20240920195230.png]]