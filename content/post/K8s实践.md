---
title: K8s实践
author: KC
type: post
tags:
  - K8s
date: 2024-05-20
description: 
toc: true
draft: false
---
1. 所有命令要主要对应的namespace
2. "PENDING" 状态表示服务正在等待底层网络资源的分配。通常情况下，当你在本地测试集群时，LoadBalancer 类型的服务会处于 Pending 状态，因为本地集群通常没有外部负载均衡器可供分配。
3. 代理会影响本地kind集群的请求