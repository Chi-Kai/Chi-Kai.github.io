---
title: "MCP 工具元数据成了攻击面：无访问审计与 tool poisoning"
date: 2026-11-04T10:35:00+08:00
draft: false
series: ["Agent Infra 随笔"]
tags: ["MCP", "tool poisoning", "MCPSEC", "OpenTelemetry", "提示注入", "Agent Observability"]
summary: "MCPSEC 只凭工具注册元数据在 177 个工具上召回 98.9% 的间接提示注入；另一篇用 STRIDE/DREAD 指出 tool poisoning 是客户端侧最普遍的洞；Claude Platform 的 OTel MCP/工具 trace 把运行时可见性接上。"
---

MCP 工具一注册，name、description、参数 schema 就同时进了客户端和模型的上下文。这段文本以前多半当「给人看的说明」；现在它本身就是攻击面，既能让人只凭元数据推断间接提示注入，也能把恶意指令嵌进描述里喂给模型。

## 只有元数据也能猜洞

闭源服务、远程托管、商业 gated、现场关键系统，第三方审计员既拿不到系统访问，也做不了动态交互。[No-Box Vulnerability Analysis](https://arxiv.org/abs/2609.10854) 把这种处境叫 no-box：手里只有功能元数据。元数据约束「符合这份约定的实现」长什么样，输入、输出、副作用都在里面，分析者可以先假设跨所有可能实现都成立的漏洞，等以后拿到更多访问再验证。

原型系统 MCPSEC 就干这件事，扫 MCP 服务器注册时暴露的工具元数据，找间接提示注入。评测覆盖 20 个常用 MCP 服务器、共 177 个工具，人工确认 95 个有洞。MCPSEC 标出 143 个可疑工具，每个附上假设漏洞与利用手法；人工确认的 95 个里，只靠元数据召回了 94 个，召回率 98.9%。同样设定下 LLM baseline 是 80/95，约 84.2%。这些数字我只对照了 arXiv 摘要页，假阳性怎么筛、143 里多出来的那些怎么标的、哪些工具被误标，我没逐条复现。

「无访问」在这里不是修辞。注册瞬间暴露给客户端的那份描述，已经够支撑一整条审计假设链。工具还没被调用，洞的形状已经能写下来了。

## 同一段描述也是投毒入口

[MCP Threat Modeling and Analyzing Vulnerabilities to Prompt Injection with Tool Poisoning](https://arxiv.org/abs/2603.22489) 用 STRIDE（Spoofing、Tampering、Repudiation、Information Disclosure、Denial of Service、Elevation of Privilege）和 DREAD（Damage、Reproducibility、Exploitability、Affected Users、Discoverability）扫了五个组件：MCP Host/Client、LLM、MCP Server、External Data Stores、Authorization Server。结论里写得很直：tool poisoning，也就是恶意指令嵌在工具元数据里，是最普遍、影响也大的客户端侧漏洞。过去 MCP 安全讨论多偏 server 侧，这篇补的是客户端这块空白。

他们拿 7 个主流 MCP 客户端做了 tool poisoning 防御对比，摘要说多数因为静态校验与参数可见性不足而存在问题。哪几个挡住了、测试用例长什么样、参数可见性缺在哪一层，摘要页没展开，我没验证。给出的防御是多层：静态元数据分析、模型决策路径跟踪、行为异常检测、用户透明机制。MCPSEC 那种「只看元数据」正落在第一层；poisoning 说明同一份注册文本既是审计输入，也是投毒入口。客户端若不做静态校验，也不把完整参数摆给用户看，模型就在用户看不见的描述上做决策。

## 调用之后要看得见

静态审注册描述不够。工具一旦被选中执行，还得看见模型调用了什么、MCP 路径上发生了什么。[Agent observability on the Claude Platform](https://claude.com/resources/articles/agent-observability-on-the-claude-platform)（2026-10-14）说 Managed Agents、Messages API 与 Anthropic SDK 可以导出遵循 OpenTelemetry GenAI 语义约定的 trace，覆盖 agent turn、模型调用、工具使用与 MCP server 调用。Console 里配好 destination，延迟、token、各类 ID 就进你的观测栈，SDK 侧可自动导出。点名接得上的有 Grafana Cloud、Honeycomb、Sentry、Splunk、Arize AI、Braintrust、Datadog、Langfuse、LangSmith by LangChain、New Relic、PostHog，也收标准 OTLP endpoint。目前是 public beta。

审过元数据、客户端挡过 poisoning，运行时若看不见 MCP/工具轨迹，多层防御里的行为异常检测和事后审计仍然缺数据。OTel GenAI 约定的用处是 Claude 与其他提供方的 trace 能进同一套仪表盘，不为 MCP 调用另开一套私有格式。发布里有一句很贴这条线：信任 agent 拿到更多权限，起点是先能追踪它的模型调用、工具使用和 MCP 调用。

同一份工具元数据因此同时扮演三种角色：给模型选工具用的说明书、给审计员做 no-box 假设的唯一输入、给攻击者塞指令的投毒面。注册协议若把 description 当自由文本、客户端又原样拼进 prompt，这三种角色就会叠在同一段字符串上。MCPSEC 证明「只读这段字符串」已经能高召回地标出间接提示注入形状；tool poisoning 论文证明客户端多数还没把这段字符串当不可信输入来校验；Claude 的 OTel 导出则把「这段字符串被选中之后发生了什么」拉进可观测范围。

三条串起来：工具描述与 schema 本身是攻击面；注册时静态审计（MCPSEC 那种 no-box）加上客户端防 poisoning，再加上运行时 OTel 级的 MCP/工具可观测，才是一条完整的线。哪些客户端具体防住了、MCPSEC 假阳性细节，我只读了摘要/论文页，没验证。

还没想清楚的几件事：

- 注册元数据若允许服务端热更新，客户端何时重审，是否把 diff 暴露给用户。
- 静态判「可注入」与运行时「真被利用」之间，OTel span 要留哪些字段才够做关联。
- 用户透明机制具体长什么样，是展示完整 description 还是只标高危字段，公开材料里我没看到统一做法。

本文由 AI 辅助整理
