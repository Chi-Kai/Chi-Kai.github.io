---
title: "Durable 不只是意图日志：event history、语义感知 C/R 与 resume 时钟"
date: 2026-11-05T10:45:00+08:00
draft: false
series: ["Agent Infra 随笔"]
tags: ["Agent Executor", "Temporal", "Crab", "durable execution", "checkpoint", "Agent Substrate"]
summary: "Google Agent Executor 用 event log + snapshot 给任意 actor 做 durable execution；Temporal Agent Harness 把内层 harness 包成 Workflow，轨迹落成 AgentEvent；Crab 用 eBPF 看 turn 级 OS 副作用决定 checkpoint 粒度，恢复正确率约 8%→100%；Beyond 49ms 提醒测的是 TTUW，不是节点本地 resume。"
---

长跑 agent 要能断点续跑，光有会话文件不够：还得有一份可回放的执行历史，以及与历史对齐的沙箱状态。Google 在 [Agent Executor](https://cloud.google.com/blog/products/ai-machine-learning/agent-executor-googles-distributed-agent-runtime)（2026-05-20）里把这件事做成了开源 runtime：任意 actor，agent、harness、skill、tool、sandbox，都靠 event log 加 snapshot 做 durable execution，进程挂了、HITL 卡着，也能从日志续。同一套能力里有单写者会话一致性，用来压住分布式流程里多人同时改 session 的风险；客户端断线后可以按 last sequence 回填，少丢一段响应；checkpoint 还允许 trajectory branching，同一条轨迹分叉试不同决策而不丢上下文。

配套的 Agent Substrate 面向「百万级亚秒工具调用」。标准 K8s 控制面吃不下这种 chatter，Substrate 用更薄的控制面把 agent 在 ready 算力上搬上搬下，仍挂在 Kubernetes 生态里，声称可支撑上亿级已注册 agent，并把已有沙箱的安全运行时与 snapshot 能力接到这层控制面上。Agent Executor 自称 harness-agnostic，可跟 Antigravity、Managed Agents、ADK、A2A 等混部，企业侧也能把 MCP、skills 和自建 agent 跑在自己的 data plane 上；目前 preview。

业务侧已经选好内层 harness 的，更常见的落点是在外面再包一层。[Temporal Agent Harness](https://temporal.io/blog/temporal-agent-harness-durable-agent-infrastructure)（2026-08-20）不重写 agent loop，而是把现有 OpenAI Agents SDK、PydanticAI、Gemini 包成 Temporal Workflow。Durable 在这里不是某个功能的补丁：工具审批、turn 编排、callback tool、code mode 沙箱，都建在同一份 Event History 上。Worker 崩了、部署滚动了、模型或工具按策略重试，都可以从离开的位置接着跑，已完成的工作不重放。

Harness 引入了比 ReAct 循环更高一层的 turn，一次 turn 调用内层 harness 恰好一次，内层 loop 可以跑到终端态；客户追问换地址、审批到达、定时器触发，都算新 turn，上下文由外层续。Tool 侧留了一道 policy 缝：模型决定用能力与能力真正执行之间可以挂审批，默认策略示例是 `ToolApprovalPolicy.allow_inherently_safe()`，策略可运行时改、可按工具或会话收窄。Code mode 让模型写一段编排多工具的 Python，这段程序仍要服从同一套审批与可观测，沙箱边界封上并不等于可以当黑盒。轨迹落成结构化 AgentEvent 流，turn、模型交互、工具调用与结果、审批、handoff，前端、审批服务、审计、评测可以独立消费同一条流，也能按事件回放，而不是让模型事后「回忆」发生了什么。不管内层是 OpenAI Agents SDK 还是 PydanticAI，外层看到的 AgentEvent 模型一样。项目定位写得很早：早于 public preview，API 会变。

Event history 保住了「做过什么」，沙箱里装过的包、挂着的进程，聊天记录接不住。[Crab](https://arxiv.org/abs/2604.28138) 盯的是 agent–OS semantic gap：框架看得见 tool call，看不见 OS 副作用；OS 看得见状态变化，没有 turn 语义来判断哪些变化跟恢复有关。论文量化过稀疏性：超过 75% 的 turn 不产生 recovery-relevant state；Terminal-Bench 轨迹里 `run_shell_command` 占工具调用约 60.4%，命令副作用又几乎无法从语法推断。

轻量恢复不够用：chat-only 在 Terminal-Bench 上大约只有 8%–13% 正确（iFlow-cli / Claude-code），chat+FS 提到 28%–42%，仍会漏掉长活进程。另一端每 turn 全量 checkpoint 在 64/96 共置密度下可把 Claude-code 拖到 3.06× / 3.78×。Crab 做 host-side runtime，不改 agent、不换 C/R 后端。eBPF Inspector 在 turn 边界看文件系统与进程净变化（含 soft-dirty 页），把粒度分成跳过、仅 FS、仅进程、全量；Coordinator 把 checkpoint 叠进 LLM 等待窗口，响应回来时若 dump 未完再 gate，并把已暴露延迟的任务提到高优先级队列；host 上的 Engine 用 ZFS + CRIU（经 runc）拼出版本化 manifest，只更了文件系统时会把新 FS 工件与上一份进程工件配对成可恢复点。Terminal-Bench / SWE-Bench 上 Crab 恢复正确率到 100%；最多 87% 的 turn 被标成无需 checkpoint；带一次注入崩溃时端到端仍落在无故障执行时间的 1.9% 以内。这些数字我对照了 arXiv 摘要与 HTML 全文，本地没复现。

快照与 resume 的数字本身也容易测偏。[Beyond 49 ms](https://ajing.github.io/posts/2026-08-14-beyond-49ms-agentic-rl/) 把「49 ms resume」拆成几只时钟：Backend-ready（节点本地 restore 到 envd 就绪）、Resume RPC、Service-ready、TTFC（第一条命令跑完）、TTUW（有用工作被 oracle 校验通过）。AgentENV / Kimi K3 报过最低 49 ms resume、133 ms 增量 checkpoint，但官方 snapshot benchmark 走的是节点本地路径，不经过网关、调度、用户命令与回传；并发基准把 50 worker × 10 次压成 10 个均值点，推不出单请求 p95/p99。作者自己的 `beyond-49ms` harness 能记这些时钟，正式 KVM 数据集还没跑出来，文中写明 not yet measured。结论收得很紧：测的是有用工作时间，不是节点本地 restore；投机执行可以读 ahead，提交前仍要验证 action 与 origin state，同一条 `pytest` 在不同文件版本上结果不同，可复用单位是 `(action, origin state)`。

四条落在同一条线上：Agent Executor 把 event log + snapshot 做成 runtime 标准；Temporal 用 Workflow Event History 当外层 harness 的地基，并导出可回放的 AgentEvent；Crab 在 turn 边界用 OS 可见副作用决定要不要、要哪种 checkpoint，把正确恢复从聊天记录那一档拉到全量沙箱态；Beyond 49ms 提醒 resume 广告数字只覆盖管道里的一截。还没想清楚的几件事：

- Agent Executor 的 event log 与 Temporal Event History 字段是否需要对齐，公开材料里我没看到互操作规范。
- Crab 的 Inspector 对「写了又删」会假阳性成 FS change（正文 FPR 约 2.3%）；这类误触发在更高密度下会不会把跳过率从 87% 啃回去，论文没给上限。
- Backend-ready 到 TTUW 的缺口，在共置 resume 爆发时主要耗在哪一层，Beyond 49ms 的 KVM 基线还没出来。

本文由 AI 辅助整理
