---
title: "快照带走了什么：分层状态、memfile 里的 token、以及根本不进 guest 的密钥"
date: 2026-11-02T20:00:00+08:00
draft: false
series: ["Agent Infra 随笔"]
tags: ["Directory Snapshots", "Firecracker", "memfile", "凭证", "Tracekit Box", "Modal", "Timbre"]
summary: "Modal 把沙箱状态拆成可单独挂载的目录快照；Timbre 用 strings 在 Firecracker memfile 里数出明文 token；Tracekit Box 让密钥只在 host 按绑定 host/header 注入，guest 只见占位符。"
---

沙箱「做完快照就算持久化」这句话里藏着三件不同的东西：可恢复的项目状态、跟着内存文件走的凭证、以及根本不该进 guest 的密钥。把三件事拆开，选型时才不会拿整盘镜像去回答密钥问题。

## 四层文件与一个目录

Modal 在 [Directory Snapshots](https://modal.com/blog/directory-snapshots-resumable-project-state-for-sandboxes) 里把沙箱文件系统拆成四层：base OS、平台依赖、应用代码、生成物。四层变化节奏不一样，系统库跟安全补丁，依赖跟工具链，应用代码跟任务，生成物常常用完就扔。整盘快照把它们捆成一份原子镜像；Directory Snapshot 只拍某一个目录，挂进另一台沙箱，并且可以和更新过的 base image 组合。补丁或 harness 升级不必整盘重建，项目目录里跑出来的状态也不用丢。

热池场景更直接：池子里先放一批相同的通用沙箱，认领到具体项目后再 `mount_image` 挂上 `/project`，预热生命周期与项目状态就此分开。Modal 文档把三类快照的适用面写清楚了，Filesystem 做完整备份或定制 base，Directory 管分层与冷启动，Memory 保进程内存态。Directory Snapshot 在最后一次使用后保留 30 天。挂载时内容相对整盘镜像优先预加载，把常用的项目代码或虚拟环境单独拍进目录，大 base image 不会拖死首批文件的可见延迟。

目录级快照解决的是「哪些文件该续上」。凭证问题出在另一层。

## strings 数出 65 次明文

Firecracker 官方 snapshot 文档写明 cryptographic tokens 会随快照复制，并把 host、API 通道、快照文件本身放进受信边界；快照落到不该到的地方，责任在集成方。Timbre 的 [Credentials in the Snapshot](https://timbreai.substack.com/p/credentials-in-the-snapshot) 在开了 nested virt 的 EC2 上亲手验证了这件事：把 token 放进 kernel cmdline，暂停后做 Full snapshot，对 256 MB 的 memfile 跑 `strings | grep`，同一明文出现 65 次，分布在 cmdline、init 环境、dmesg、journal 四处。没有 CVE，文档警告的就是这个行为。

memfile 是 guest RAM 的原始拷贝，恢复时 `MAP_PRIVATE` 做写时复制，没有加密层，也没有 redact 过滤器；完整性只有 VM state 文件上的 64 位 CRC，挡的是误损坏。

四家平台的放法各不相同。E2B 的 MMDS 只放 token 的 hash，真值在 resume 后由 host 经 HTTP POST 注入 guest 里的 `envd`；token 仍会进 `envd` 的堆，快照若在注入后拍，memfile 里照样有明文。Modal 建在 gVisor checkpoint 上，另有 Secret 原语在 attach 时注入；应用代码若在快照前把 secret 读进模块全局，值就进了快照，暴露窗口取决于客户代码何时读、何时丢掉引用。Fly Sprites 声称 checkpoint 只搬 metadata，可 Sprite 底下仍是 Firecracker，host 上的 memfile 还在，对象存储里的磁盘块是另一处暴露面。AgentCore 写会话结束后 memory sanitized，公开文档没说清 sanitize 具体怎么做，是否清零底层页，我没找到依据。

Timbre 列了几条不依赖硅片的减面办法：late-binding、短时凭证、注入引用而非明文。SEV-SNP / TDX 能挡住宿主机读内存，挡不住 guest 内被 prompt injection 直接外泄；硬件加密回答的是「host 能不能读」，对「agent 自己会不会把 token 发出去」帮不上忙，而后者在 agent 工作负载里往往是主威胁面。

## 密钥停在门外

密钥若根本不进 guest，memfile 里就抄不到。Tracekit Box 论文 [Running AI Agents in Micro-VMs Where Every Way Out Is Decided and Signed](https://arxiv.org/abs/2610.38458) 给每个任务一台无网卡 Firecracker microVM，出口只剩 host 上的 vsock 门：命令、出站、密钥、模型调用、workspace patch。门上由 agent 够不到的 signer 做决定，签进可离线校验的 `.tkb` bundle。

密钥只在 host 按绑定 host 与 header 注入；guest 里只有占位符，host 代理在去往绑定主机的那一个 header 上做替换，别处出现占位符或真值就拦，响应里也会 redact。DNS 在 host 侧解析，私网、链路本地、云 metadata 地址一律拒绝，跟 allowlist 怎么写无关。workspace 构建时不拷 `.env*`、密钥文件一类路径；改动按文件经 signer 后再写回 checkout，`git apply` 不直接打在用户工作树上。

已知限制写在摘要里：v0 未上 jailer，记门不记 guest 内进程。KVM 上的 e2e 13 项检查全过，含 pull 与 boot 约 28.5 秒；仓库 demo 导出的 bundle 有 102 条已签名记录，其中 46 条决策。没有 signer，daemon 直接起不来；必须留下的观察写不进去，盒子会停，损失记进 `tkbox.gap`。沙箱管得住出界却少证据，追踪能签名却管不住没走到 hook 的动作，Box 把这两半接在同一组门上。

三件事叠在一条线上：目录快照让可恢复的状态分层带走，memfile 让进程里曾持有的凭证跟着整份内存走，无网卡门控让密钥不必进那份内存。密度与启动开销是另一条尺子，这一期不展开；若要拿真实构建曲线定规格，[Directionally Local](https://directionallylocal.com/blog/what-a-coding-agent-sandbox-really-needs/) 用 OpenCode 的 clone / install / typecheck 在自托管 Firecracker 上测过相对裸机约 16% 开销，只能算旁证。

还没想清楚的几件事：

- Directory Snapshot 与 Memory Snapshot 叠用时，Secret 的注入时机该卡在哪一步，才不会把刚注入的值拍进 memfile。
- AgentCore 的 memory sanitized 在实现上是否清零底层页，公开材料里我没找到。
- Tracekit Box 记门不记 guest 内进程：一条经 signer 的 `exec` 再 fork 出一百个子命令，门上只留下一条 Bash 决策，中间泄漏路径要靠别的门补上。

本文由 AI 辅助整理
