---
title: "沙箱规模化：不养热池的控制面，与亚秒级状态搬迁"
date: 2026-11-06T10:30:00+08:00
draft: false
series: ["Agent Infra 随笔"]
tags: ["Agent Substrate", "gVisor", "Firecracker", "DeltaBox", "sandbox", "checkpoint"]
summary: "Google 把 Agent Substrate 与 gVisor 捐入 CNCF Sandbox，用 actor/worker 解耦空闲占位；Adapt 的 orc 用 Firecracker 按需冷启、不养热池；DeltaBox 用 DeltaFS+DeltaCR 把 checkpoint/rollback 压到约 10.8/1.9 ms。"
---

标准 Kubernetes 装不下 agent 会话的空闲期：Pod 整天热着，一万活跃用户付不起；serverless 缩到零，冷启要重新 clone、装包、重建内存，唤醒超过 20 秒，交互就断了。Google 在 [Sandboxing Untrusted AI Agents at Scale](https://opensource.googleblog.com/2026/11/sandboxing-untrusted-ai-agents-at-scale.html)（2026-11-05）把这条缝写成目标：完整内存与磁盘状态下，唤醒压进 500 毫秒以内，空闲期不付热 Pod 的账。KubeCon NA 前夕，Agent Substrate 与 gVisor 一起捐进 CNCF Sandbox，同批还有 OpenRL。

Agent 大部分时间在等模型吐 token 或等人审 PR，算力几乎不用，内存和磁盘上的 Git 仓库、语言运行时、未提交改动却不能丢。平台要的是带齐状态的亚秒唤醒，而不是再养一池热 Pod。

Substrate 把 Actor 和 Worker 拆开。Actor 是一条会话的状态容器，文件系统、环境变量、内存快照、网络规则都挂在它上面；Worker 是 WorkerPool 里预热好的 Kubernetes Pod，只提供容器运行时和隔离边界。一次 turn 进来，atenet-router 先把请求停在内存里，控制器把 Actor 塞进空闲 Worker 槽，从分层存储拉回内存和磁盘，进程醒来，代理放行请求，官方说法是不到 500 毫秒回到暂停点。turn 结束就快照 Actor、把 Worker 槽还回池子。Actor 状态不靠新建 Pod 对象维持，每秒成千上万次激活与挂起才不会砸 etcd。训练和评测的场景正好相反：上千个短命沙箱同时起落，每个都做成完整 Pod 的话，kube-apiserver 和子网 IP 先顶不住，Substrate 把沙箱分配从「管稳态拓扑」挪到更薄的一层。

隔离侧照样要处理模型吐出来的未验证代码。gVisor（runsc）在用户态拦系统调用，自己维护内存与进程表，Substrate 才能把环境冻住、把页写到存储、再在另一台节点上恢复，同样卡在亚秒。需要硬件虚拟化或自定义内核模块时，同一套 API 下挂 Cloud Hypervisor microVM。这些数字我没在本地复现过，只对照了博客正文。

控制面不想养热池，另一条路走得更绝。Adapt 的 [How we run sandboxes for agents at scale](https://adapt.com/blog/orchestrating-agent-sandboxes)（Sean Smith，2026-07-03）里，每个对话对应一台独立机器，模型以 root 装软件、写程序、打 API。最早在 GKE Sandbox 上跑 gVisor，Dockerfile 定义底座、靠 GKE 扩缩，上路很快；问题出在 gVisor 用户态重实现 syscall，模型和 I/O 重的活会踩兼容与性能坑，启动时间、装箱密度、回收策略又被托管调度器卡住。迁到 Firecracker microVM 之后，他们自建控制面 `orc`：创建时把 OCI 镜像转成 rootfs 并缓存，底座仍是 Dockerfile，需要 Postgres+pgvector 之类时直接指另一张镜像。Firecracker 冷启在几百毫秒量级，快到他们干脆不养热池，聊天开始就建、安静就拆，没有闲置池可付账。密度靠给每台 microVM 配它真正需要的 CPU/内存，而不是按容器习惯超配。`orc` 的 API 很窄：按镜像建 VM、灌命令、读写文件、打标签、删掉；客人里跑一个小 init 当 PID 1，各自独立网络。

Substrate 管的是会话挂起再唤醒；树搜索和 RL 还要更密的 checkpoint/rollback：同一份沙箱状态要在父子节点间来回切，那是「挂起唤醒」覆盖不到的方向。文件系统和进程内存必须成对一致，聊天历史截断解决不了 `rm` 和 `pip install`。上海交大等团队的 [DeltaBox](https://arxiv.org/abs/2605.22781)（arXiv:2605.22781）看的是另一件事：相邻 checkpoint 之间往往只差一点点，全量拷贝整盘整内存才是瓶颈。他们提出 DeltaState，DeltaFS 在不卸载的前提下热切换 overlay 层，冻结当前可写层、插入新层，更新走 copy-on-write，回滚变成 O(1) 换层；DeltaCR 每次 checkpoint 做一次 CRIU dump，同时 fork 出冻结模板，回滚优先从模板 fork，模板被驱逐再走 CRIU 慢路径。论文在 SWE-bench 与 RL 微基准上给的数是：checkpoint 本地工作约 10.83 ms，藏进推理等待窗口，agent 感知阻塞约为 0；模板命中时 rollback 约 1.86 ms。相对自托管 E2B 增量快照基线，状态管理开销从总时间的 23%–48% 降到 1%–2%。这些我只读了摘要和 HTML 全文，没本地跑。

三条线落在同一问题上。Substrate 用 actor/worker 把空闲会话从热 Pod 里拆出来，亚秒恢复靠 gVisor 与 Cloud Hypervisor 的统一快照 API；Adapt 的 `orc` 靠 Firecracker 冷启快到可以不养热池，OCI→rootfs 保住 Dockerfile 工作流；DeltaBox 把「相邻状态几乎一样」做成变更式 C/R，把树搜索关键路径上的回滚压到毫秒。

还没想清楚的几件事：

- Actor 外置状态与 DeltaBox 的进程级模板池能不能叠在同一条会话链上，公开材料里我没看到互操作说明。
- Adapt 不养热池，前提是「几百毫秒冷启可接受」；交互编码助手如果把 TTUW（有用工作就绪）也算进时钟，这条线是否还成立，文中没给端到端数字。
- DeltaBox 的 Network Proxy Daemon 把 LLM 连接挪出 agent 地址空间，模板才敢安全 fork；网络侧副作用目前不回滚，树搜索碰到外部 API 写操作时边界怎么画，论文留了口子。

本文由 AI 辅助整理
