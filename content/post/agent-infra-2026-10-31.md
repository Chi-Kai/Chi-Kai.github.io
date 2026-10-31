---
title: "Agent 计算机要可休眠、可分支：Sprites、Firecracker 与 Dogwood"
date: 2026-10-31T20:00:00+08:00
draft: false
series: ["Agent Infra 随笔"]
tags: ["Sprites", "Firecracker", "Unikraft", "Strands Box", "Dogwood", "Sandbox"]
summary: "Fly Sprites 用秒级创建、持久根盘和空闲廉价休眠重做编排；Plori 在嵌套 KVM 上冷启动 305 个 Firecracker，create→健康 agent p50 1.38s；Unikraft 把外部 CoW 拷贝叫 branch、guest 发起才叫 fork；Strands Box 用内核收束出口、Dogwood 写语义策略。一条线：agent 要可休眠可分支的计算机，隔离边界不能只挡 syscall。"
---

Agent 要的计算机形状很具体：秒级创建、根盘能留下来、闲着就睡到几乎不花钱，还得能从一份状态分出另一份去试。Fly.io 的 [Sprites](https://fly.io/blog/design-and-implementation/) 把这四条写成产品约束：Linux VM、给你 root、创建一两秒、100GB 持久根盘、空闲自动休眠且休眠几乎不花钱。为做到这一点，他们重做了编排栈，推翻了 Fly Machines 的若干核心决策。

Machines 的 start/stop 可以很快，但 create 本身可能超过一分钟，靠的是事先建好再停着待命。Sprites 要的是 create 本身快到像机器早就在那儿：丢掉「每台机器拉一份用户容器镜像」这条路径，改成标准容器、工作节点预先知道内容，创建不再卡在 pull/unpack。根盘也不再把 NVMe 当存储根，而是对象存储上的 blob，NVMe 只做读穿缓存；持久状态实质是一个 URL，机器可以挪，编排不必绑死在某块本地盘上。编排重心也翻了：多数管理工作进 guest 里的 root 命名空间，用户代码跑在夹层容器里，checkpoint/restore 不必整机重启。Checkpoint 做得够快，他们希望当成基础能力用，而不是偶发备份；预装的 Claude Code 会主动做 checkpoint。Fly 自己也写明：Sprites 优化的是交互式、可睡的用法；电商那种按毫秒量级保暖接请求的 workload，更贴近 Machines 那套语义。

「秒级 create」在嵌套虚拟化上也能跑出生产数字。Plori 把节点本身放在 KVM guest 里，从 gVisor pod 切到嵌套 KVM 上的 [Firecracker](https://plori.ai/blog/firecracker-sandboxes-nested-virtualization)。2026-09-25 起切流量，到 09-27 约 19:38 UTC，舰队冷启动了 305 个 microVM；create 请求到健康 agent 进程 p50 1.38 s、p95 1.76 s，其中 Firecracker 配置并启动 guest 的中位数约 10 ms，大头在 guest 里 agent 进程起来。节点规格是 2 vCPU / 8 GB / 120 GB 本地盘，Ubuntu 24.04、内核 6.8、`kvm_amd` 开 nested；agent guest 用 2 vCPU / 1024 MiB，钉 Firecracker v1.17.0 和 6.18.44 guest 内核。节点上有个小 daemon 经 HTTP 创建/停机，guest init 拉配置再起 agent。耗时拆成 create（含网络槽、根盘 clone、jailer）、boot（配置到 InstanceStart）、health（到 agent 健康检查成功）三段；生产 305 次冷启动里 boot 中位 10 ms，health 中位 1.34 s。文档里两种校时方式在这批主机上失败，他们改走 NTP，用节点的 chronyd 给 guest 校时。嵌套虚拟化没把冷启动推到秒级以外；对「节点本身也是 VM」的部署来说，这组数字说明 create 预算主要不在 hypervisor 那十几毫秒。

机器能快起、能带着盘睡，下一步是从一份活状态分出另一份。Unikraft 在 [branch 不是 fork](https://unikraft.com/blog/branch-is-not-fork) 里把 sandbox 圈常混用的「fork」掰开：平台从外部对运行中 microVM 做 CoW 快照拷贝叫 **branch**，两个对等实例互不感知；真正像 `fork(2)`、由 guest 发起、父子双方通过返回值知道身份的才叫 **fork**。Branch 用 `unikraft run --branch`，内存和卷 CoW，源实例暂停毫秒级；文中写到极端情况下分支一台 1TB Postgres 的 microVM，操作落在 100–200 毫秒。常见密度下 branch 大约十毫秒。Guest 侧 fork 走 `/uk/libukp/fork`：父进程拿到子 UUID，子进程读到全零 UUID，子实例挂在 `/uk/libukp/vm/<uuid>/` 下，父侧能跟踪自己扇出的实例。起子 agent、拷一份 prod 库做开发、预览环境用完丢掉，走的是 branch；要 worker pool、supervisor 扇出子 agent 并跟踪每一个，才需要 guest 侧的 fork 契约。名字一旦写成 fork，调用方会按 Unix 语义去设计；叫错了，API 契约会带着人走进死胡同。

机器能睡、能分支，出口策略仍可能卡在 syscall 层。Marc Brooker 写的 [Strands Box](https://strandsagents.com/blog/strands-box-the-big-picture/) 把内核强制隔离用成收束器：OS 机制只保证「出箱只有一条路」，真正策略用 Dogwood 写在用户态，覆盖 shell、Python、MCP、文件系统和网络。例子很直：「测试通过才允许 git push」，或「payments API 一天最多 100 美元」。Agent 要动态探索，边界又要确定性执行；在 open/connect 上写 allow/deny，表达不了「测试绿了才能 push」这种工具语义。Syscall 过滤还要跟着内核接口涨，allow list 兼容性、deny list 逃逸面都不舒服；gVisor 把 syscall 模拟掉了，语义仍到不了 MCP 调用这一层。Box 的代价也写在原文里：用户态拦截器代码面更大，绕过策略的 bug 面也更大；多租户生产他们建议外面再套 MicroVM（EC2 / Lambda / Fargate 一类），本地用 Box 的语义粒度，云端再用更硬的 VM 边界。微软的 Agent Governance Toolkit 把 MCP 调用的允许检查放到协议外的控制面，SandboxEscapeBench 用 CTF 形态测模型逃出 Docker/OCI 的能力；这两条今天不展开，只提醒：协议标准化了发现与调用，不等于内建了「这次能不能做」。

还没想清楚的几件事：

- Sprites 的休眠与 checkpoint 对交互式 agent 很顺；要 millisecond 级唤醒接 HTTP 的 workload，Fly 自己也说更适合 Machines。同一控制面要不要两套休眠策略，运维上怎么切？
- Unikraft 的 100–200 毫秒来自原文 1TB Postgres 的极端例子；换自己的镜像和卷布局，延迟会漂到哪，我没验证。
- Dogwood 写在用户态，策略表达力上来了，TCB 也变厚了。本地 Box + 云端 MicroVM 的分层说得通，分界运维成本我猜不低。
- 嵌套 KVM 上 Firecracker 能跑到 p50 1.38 s，校时却要改 NTP；下一批硬件或 hypervisor 版本，哪些「文档写法」还会失效？

本文由 AI 辅助整理
