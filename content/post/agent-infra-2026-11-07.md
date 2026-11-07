---
title: "本地 coding agent 沙箱：能用才算安全"
date: 2026-11-07T10:35:00+08:00
draft: false
series: ["Agent Infra 随笔"]
tags: ["SideKernel", "smolvm", "Docker Sandboxes", "Harbor", "Tarit", "microVM"]
summary: "SideKernel 的调研里 38.2% 受访者在沙箱里跑 coding agent，23 项能力测试把可用性写成硬指标；Simon 实测 smolvm 1.8.3 冷启 0.6–1.5s；Harbor 给每个评测任务一台 Docker Sandbox microVM 并按阶段切网络；Tarit 把 live snapshot 与有界冷启池放进同一套 KVM 栈。"
---

本地 coding agent 要 root、要装包、要改仓库，同时又是不可信组件。Georgia Tech 的 Dimitrios Prasakis 在 [SideKernel](https://arxiv.org/abs/2610.02456) 里把这个矛盾做成了可量化的东西：在线调研的 68 名 coding agent 用户中，38.2%（26 人）当前在沙箱里跑，10.3%（7 人）曾经用过。调研把主机与沙箱文件共享、网络端口、环境准备列成主要可用性障碍；作者据此把「能用」写成设计目标。

论文据此做了一套基于 Apple Virtualization 的 macOS microVM 沙箱，把调研里暴露的可用性障碍翻成 23 项能力测试，再拿同样筛选条件下的方案逐项对比：Docker Sandboxes、SmolVM、Microsandbox、Shuru、VibeBox 都在表里。作者写，可用性相关能力上 Docker Sandboxes 与 SideKernel 得分最高。分项粗看，UB-1（主机与沙箱文件共享）九项里 SideKernel 过八项、Docker Sandboxes 过六项，SmolVM 一类过五项左右。

威胁面被拆成南北两条边界。南界是沙箱对主机的通道：文件、端口、凭证，要挡住装恶意软件、删改主机文件；北界是沙箱对外网和其他主机的通道，要挡住把已挂载数据外泄，也不让里面的 agent 对外面造成伤害。南界放了一个小 credential proxy，Claude 一类凭证不进 guest，由代理注入 API 请求；北界是 NAT kill switch，可以关掉外网、同时保留推理流量。文件侧用 `sk-drop <host-path>` 在运行时挂主机路径，需要显式批准。作者自己标注这份调研是 formative signal，不宜当统计总体。

数字更硬的一份记录来自 Simon Willison 的 [smolvm 笔记](https://simonwillison.net/2026/Aug/19/smolmachines-untrusted-sandbox/)。他让 Claude Code for web 去测 smolvm 1.8.3，任务是不信任的 Python/JS 数据变换：离线本地镜像、默认无网、CPU 与内存限额、guest 超时、存储配额、只读输入挂载、可写输出、`--unprivileged`，逐项按预期工作；冷启动 0.6–1.5 秒，热执行约 50 毫秒。测到一半卡住：Claude Code 的容器本身就是 Firecracker guest（Linux 6.18.5-fc-v20，4 vCPU / 15GB），没有 `/dev/kvm`，也没有 vmx/svm 标志，嵌套虚拟化起不来，`smolvm machine run` 直接报 kvm not available。他改走 GitHub Actions 的 ubuntu runner 跑完测试，再把 workflow 从最终提交里拿掉。SmolVM 也出现在 SideKernel 的对比表里；这两份材料叠起来说明一件事：硬件隔离沙箱若默认宿主机有裸机 KVM，嵌套部署时连测都测不动。

评测场景把同一问题放大上千倍。Docker 的 [Harbor + Docker Sandboxes](https://www.docker.com/blog/harbor-docker-sandboxes-run-safe-reproducible-agent-evals/) 分工是：Harbor 跑任务、打分、收日志，Docker Sandboxes 给每个任务一台 microVM，自带内核，agent 碰不到宿主机也碰不到别的任务。agent 会装包、改文件、跑命令，也会 reward hacking，比如改测试或覆盖记分文件拿分。Cloud Sandboxes 在 setup 阶段放开网络下依赖，agent 真正开跑时再收紧出口，压跨任务污染和外网捷径；同一套任务定义本地和云上都能跑，本地集成目前写的是 public 与 no-network 两种模式。文中写测过 64 路并发。这版 Compose 也进来了，多容器任务落在同一台 Cloud Sandbox microVM 内。隔离的清单因此多两条：别让任务互相看见，别让外网变成作弊通道。

单机「能用」和测评「可复现」之外，还有控制面。InstaVM 开源的 [Tarit](https://instavm.io/blog/introducing-tarit) 的出发点是：只开 Firecracker 不够，快照、热池、调度得一起设计。Firecracker 的快照文档要求先暂停 microVM 与 vCPU，也不负责在 host 上打包管理快照，放置、暖池、路由、容量、故障切换都在别的服务里。作者说这层他们单独做过不止一次，跨 API 调一项就弄坏另一项的假设，于是把 VMM 和编排放进同一仓库。`vmm` 是基于 rust-vmm 的进程-per-VM，设备模型只有 MMIO virtio，没有 PCI 和 BIOS，exec 与交互式 PTY 走 vsock；`taritd` 跨节点编排，成员与归属落在 PostgreSQL，请求打到非 owner 节点会鉴权后转发。live snapshot 用迭代 pre-copy：拷内存时 vCPU 继续跑，跟踪脏页，收敛后短暂停机拷剩余页并抓 CPU 与设备状态，目前仅支持单 vCPU；diff snapshot 只带脏页，restore 可走 `userfaultfd` 惰性入页。暖池是先冷启一台 golden VM、就绪后快照，再在后台 restore 补池，guest 内存 CoW 按需分页，盘是只读底座加可写 overlay，create 租一台已起来的机器而不是再 boot 一遍。

作者在 `c8i.metal-48xl` 上给了有界冷启池的数字：顺序创建 p95 约 38 ms，100 次突发创建 p95 约 56 ms，并注明这是单机单配置。比这组数字更有用的是一条失败案例：100 路并发冷启把 live exec 饿死，后续交错执行超过 30 秒超时。补池要有界，可以放进低权重 cgroup，否则重建容量会抢走正在干活的 agent 的 CPU。

四份材料落在同一条线上：安全边界画对了只是入场券，文件共享、端口、凭证、环境准备得像第二台电脑，采用率才会从三成八往上走；硬件隔离的冷热启动量级在毫秒到秒，但嵌套 virt 会先把测试卡住；评测把隔离要求推进到任务之间和外网出口；生产侧则要把 live snapshot 与有界暖池和 VMM 放进同一栈，不然冷启突发先砸到正在跑 exec 的会话。

还没想清楚的几件事：

- SideKernel 的 23 项能力测试测的是可用性，不是逃逸；同一套分数在 prompt injection 下会掉多少，论文没给。
- Harbor 本地集成只有 public 与 no-network，和云上「setup 放开、跑时收紧」差多少，公开材料没展开到我能复述的程度。
- Tarit 的 38/56 ms 是冷启池 create 的延迟，不是「agent 已就绪可交互」；算上 guest init 和 agent 拉起会漂到哪，文中没有对照。
- Simon 那条 GitHub Actions 绕行说明嵌套 KVM 仍是无解的现实约束，我没有验证过其它 CI runner 是否一样。

本文由 AI 辅助整理
