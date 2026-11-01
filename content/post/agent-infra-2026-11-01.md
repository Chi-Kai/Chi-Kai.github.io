---
title: "Agent 崩溃后从哪一步续跑：durable harness 与 runtime 分层"
date: 2026-11-01T15:39:00+08:00
draft: false
series: ["Agent Infra 随笔"]
tags: ["durable-harness", "Workflow SDK", "Cloudflare Agents", "Durable Object", "Pi", "PVM", "Agent Loop"]
summary: "Vercel 的 durable-harness 在每次模型/工具调用前后写意图和结果，崩溃后新函数读日志续跑；Cloudflare 把 runtime 和 harness 拆成两层，Pi 1.4 会 park 反复崩溃的任务；Armin Ronacher 提醒外层循环会放大模型的坏习惯；Karinja 的 PVM 基准给出暂停/恢复的毫秒数，但快照不保留进程。"
---

Agent 在云上崩溃后能不能续跑，取决于有没有一份记录「哪些调用已经做完」的日志，跟机器本身稳不稳关系不大。Vercel 在 [Making every harness durable](https://vercel.com/blog/making-every-harness-durable) 里点出了这个缺口：VM 加持久盘能复刻本地环境，但 VM 不会自动记下哪些模型调用、工具调用已完成，也不会在崩溃后把中断的那一轮接上。

## 意图日志加队列兜底

Vercel 开源的 `durable-harness` 是一组适配器，让 Pi、OpenCode、Codex、Claude Code 这些现成 harness 把会话存进 Workflow SDK 的持久后端，SDK 管这个后端叫 World。机制很朴素：模型调用或工具调用执行前写一条意图，执行完写一条结果；整轮再由一条队列消息兜底，函数死了，队列把这一轮交给新函数，新函数读日志接着做。

崩溃后日志能分出三种状态：做完的、正在跑的、没开始的。做完的调用直接从日志读结果，不重放，所以已完成的模型调用不会被计两次 token。被打断的工具分两类，标了 `replay: 'safe'` 的重跑；其余的以「可能已执行」交回给模型，由模型决定要不要先检查一下。

这个设计把幂等性的判断拆给了两方：工具作者声明哪些能安全重跑，剩下的交给模型。我猜 `git push`、发 PR 这类工具多半不会标 safe，模型拿到「可能已执行」后得自己去查 PR 列表，这一步做得好不好要看具体模型，我没验证过。

Claude Code 的 transcript、Codex 的 rollout、OpenCode 的事件存储这类 harness 自带的会话文件，每条记录也会镜像进日志，新函数或新 Sandbox 能据此重建。World 是一个接口，合并了 Storage、Queue、Streamer 三样；本地开发用文件，自托管可以用 Postgres、MongoDB、Redis 或 Turso，切到 Postgres 只要设两个环境变量 `WORKFLOW_TARGET_WORLD` 和 `WORKFLOW_POSTGRES_URL`。

## Runtime 管住处，harness 管每一轮

Cloudflare 的 [Harnesses 文档](https://developers.cloudflare.com/agents/harnesses/) 把同一件事切成两层，原话是 runtime 回答「where does this agent live and how does it stay durable?」，harness 回答「what does this agent do on each turn?」。harness 负责拼 prompt、加载记忆、选工具、处理流、决定继续还是停；runtime 负责启动、持久任务、路由和装上去的能力。推荐的起点是 PiHarness：Pi 跑在 Durable Object 里，对象被驱逐后再被唤醒。会话存储归 harness 管，Pi 和 OpenCode 自己维护 transcript。

Vercel 从外面给任意 harness 包一层，不改 harness；Cloudflare 让 harness 住进一个本身就会被驱逐、会被唤醒的对象里。两边落点不同，回答的问题一样：进程没了以后，谁知道该从哪接着做。

运维细节写在 [Pi 1.4 的变更日志](https://developers.cloudflare.com/changelog/post/2026-10-22-pi-harness-pi-1-4/) 里，比架构图更能说明续跑的难处。一个任务如果反复把对象搞崩，PiHarness 会把它 park 起来，不再无限重启，`pending()` 里能看到，`session.retry()` 可以手动重跑。存储出错后 PiHarness 会重开 Pi，不让后续每次调用都失败直到对象重启；唤醒任务失败的步骤按退避重试。模型流被截断或空闲，会被当成瞬时错误重试整轮。还有一个坑：Pi 1.4 把对象 SQLite 里的会话存储迁到了 1.3 打不开的 schema，升级后不能回滚。

## 外层循环会放大坏习惯

续跑能力解决的是「停下来以后怎么接上」，谁来判断该不该接是另一层问题。Armin Ronacher 在 [The Coming Loop](https://lucumr.pocoo.org/2026/6/23/the-coming-loop/) 里区分了两个循环：agent 内部的工具调用循环，和外面那层 harness 循环。外层的流程是任务进队列、机器认领、尝试、停下，再由 harness 判断是否真的结束，没结束就续同一会话、注入消息、换上下文重开，或者把任务交给另一台机器。

他说自己在在意的代码上用这种方式没什么成功经验。模型写的代码太防御、太局部，回避强不变量，用 fallback 代替让坏状态不可能出现；放进循环后每一轮再加一点防御，系统越来越难懂，看起来却越来越稳。他也承认循环在代码移植、性能实验、安全扫描上效果很好，举的例子包括他自己把 MiniJinja 移植到 Go。

这跟 durable harness 的关系是：日志和队列让任务死不掉，任务死不掉之后，判停逻辑写得差，代价也会被持续放大。

## 沙箱快照能接住多少

Karinja 的 [Giving every agent its own microVM](https://karinja.ai/blog/001-giving-every-agent-its-own-microvm) 用 PVM 在普通 x86 云 VM 里跑 microVM，给了一组生命周期基准：单台 4 vCPU、本地 XFS，50 次取中位/p95。本地快照 40/54 ms，暂停 88/118 ms，暂停后恢复 167/401 ms，从快照新建到首条命令 308/515 ms。快照放到 S3 兼容存储，API 约 1.5 s，上传完要 5.8 s。

关键限定在后面：这些快照只含文件系统，运行中的进程不会保留。我猜沙箱层能保住工作区，内存里的会话状态还得靠 harness 那层的日志补回来，两层缺一层都续不上，这个组合我没跑过。

还没想清楚的几件事：

- `replay: 'safe'` 的标注由谁维护，标错一个有副作用的工具会怎样。
- Cloudflare 的 park 阈值是多少次崩溃，文档里我没找到数字。
- Armin 说的判停问题，日志层能不能提供信号，比如同一工具连续被标「可能已执行」几次就停。

本文由 AI 辅助整理
