---
title: "Claude Code、OpenClaw、Hermes、DeepAgents：四种 Agent Harness 的设计对比"
date: 2026-07-07T01:15:00+08:00
draft: false
tags: ["Agent Harness"]
categories: ["系统设计"]
description: "从核心循环、上下文工程、记忆、技能与自我改进、子代理、安全边界到模型耦合，逐个维度拆解四个代表性 Agent Harness 的设计取舍，以及工程上该怎么选。"
---

过去一年，「agent」这个词已经被用滥了。但真正决定一个 agent 好不好用的，往往不是模型本身，
而是包在模型外面的那一层东西——**harness**。LangChain 在
[The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)
里给了一个很干净的定义：

> Agent = Model + Harness。Harness 是「除模型之外的所有代码、配置与执行逻辑」。

模型决定上限，harness 决定你日常摸到的下限：上下文怎么管、记忆放哪、工具怎么给、
权限怎么收、循环由谁驱动。这篇文章拆四个有代表性的 harness，看它们对同一组问题给出的不同答案。

## 四位选手

| | 定位 | 出品方 | 驱动方式 | 记忆载体 |
|---|---|---|---|---|
| **Claude Code** | 交互式编码 agent | Anthropic | 人驱动的终端会话 | CLAUDE.md / AGENTS.md + 记忆目录 |
| **OpenClaw** | 自托管个人助理 | Peter Steinberger（社区） | 消息网关 + 心跳定时器 | 磁盘上的 Markdown 文件 |
| **Hermes** | 常驻自治 agent | Nous Research | 多渠道网关 + cron 后台 | 三层记忆（SQLite + Markdown） |
| **DeepAgents** | 开发者用的 harness SDK | LangChain | 开发者定义的图运行时 | 可插拔虚拟文件系统 |

简单交代下背景：

- **Claude Code** 是 Anthropic 的终端编码 agent，也是「harness 与模型联合演化」路线的代表——
  LangChain 那篇文章特别指出，Claude Code 和 Codex 的模型后训练是**带着 harness 一起做的**。
- **OpenClaw** 是 Peter Steinberger 在 2025 年底发布的自托管个人助理（前身叫 Clawdbot，短暂改名
  Moltbot 后定名 OpenClaw），几个月内成为增长最快的开源 agent 项目之一；Steinberger 本人已于
  2026 年 2 月[加入 OpenAI](https://fortune.com/2026/02/19/openclaw-who-is-peter-steinberger-openai-sam-altman-anthropic-moltbook/)。
- **Hermes** 是 Nous Research 的开源 agent，把「Harness Engineering」这个概念直接产品化，
  主打自我改进与 7×24 后台运行。
- **DeepAgents** 是 LangChain 的「电池全含」harness（[langchain-ai/deepagents](https://github.com/langchain-ai/deepagents)），
  不是给最终用户的产品，而是给开发者构建自己 agent 的底座，跑在 LangGraph 运行时上。

四者面向的场景完全不同，但要回答的是同一组设计问题。下面逐个维度看。

## 维度一：谁在驱动循环

这是四个 harness 分歧最大的地方。

**Claude Code：人握着方向盘。** 核心是一个 REPL 式会话——你说一句，它跑一轮
（读代码、改文件、跑测试），回来找你。Plan mode、权限确认、hooks 都是围绕
「人随时在环上」设计的。它可以后台跑子任务，但会话的节奏由人定。

**OpenClaw：网关 + 心跳。** 一个常驻 Gateway 守护进程（默认监听 `127.0.0.1:18789`）
通过 WebSocket 统一接管所有消息渠道——WhatsApp、Telegram、Slack、Discord、Signal、iMessage。
消息进来，路由给 agent 循环处理。更有意思的是 **heartbeat**：默认每 30 分钟定时唤醒 agent，
读一遍 `HEARTBEAT.md` 里的任务清单，看有什么该主动做的。人不在场，agent 也在转。

**Hermes：默认常驻后台。** 同样是多渠道网关（Telegram、Discord、Slack、WhatsApp、Signal、
iMessage、微信、CLI 等 12+ 入口），但设计重心是 7×24 自治运行——官方推荐的部署方式之一就是
5 美元的 VPS，内存占用不到 500MB。cron 调度加上任务完成后自动触发的学习环（下文细说），
它比 OpenClaw 更进一步：不只是「人不在也在转」，而是「转的过程中还在自我修改」。

**DeepAgents：循环是你写的图。** 作为 SDK，它不预设谁驱动循环——开发者用 LangGraph
定义图结构，运行时提供持久执行（durable execution）、流式输出和人工中断点。
循环的形状本身就是产品设计的一部分。

工程上这条维度可以翻译成一个问题：**你的 agent 的「时钟」是什么？** 是人的下一句话
（Claude Code），是外部消息（OpenClaw/Hermes 的网关），是定时器（heartbeat/cron），
还是图运行时的状态机（DeepAgents）。先想清楚时钟，再谈其他。

## 维度二：上下文工程

四家面对同一个敌人：context rot——上下文窗口塞满之后，模型性能劣化。解法惊人地趋同，
细节各有取舍。

- **Claude Code**：自动 compaction（窗口快满时总结压缩）、工具输出落盘、子代理隔离上下文，
  以及 Skills 的**渐进披露**——技能只把名字和描述注入系统提示，用到时才读全文。
- **OpenClaw**：同样的渐进披露做得很典型——上下文组装阶段只注入技能的名称、描述和文件路径，
  模型判断相关后**按需读取** `SKILL.md`。数万个社区技能也不会撑爆上下文。
- **Hermes**：把「按需检索」推到记忆层——新会话不加载历史，而是用 SQLite FTS5 全文检索
  只拉相关片段。用了几个月之后，启动延迟也不随历史膨胀。
- **DeepAgents**：虚拟文件系统作为 offload 目标（中间结果写文件而不是留在上下文里），
  加上内置 summarization 和 prompt caching。

共识已经形成：**上下文是稀缺资源，一切能落盘的都落盘，一切能延迟加载的都延迟加载。**
差别只在「落到哪」（文件 / SQLite / 可插拔 backend）和「谁决定加载」（模型自己读 vs 检索器推送）。

## 维度三：记忆放哪

- **Claude Code**：`CLAUDE.md` / `AGENTS.md` 承载项目约定，外加自动维护的记忆目录。
  哲学是「记忆是给模型看的文档」。
- **OpenClaw**：最极端也最迷人——**agent 就是磁盘上的一堆文件**。`SOUL.md` 定义人格，
  `MEMORY.md` 存长期记忆，`HEARTBEAT.md` 是任务表。想改 agent 的行为？改文件就行。
  想审计它知道什么？打开文件夹看。没有数据库，没有隐藏状态。
- **Hermes**：结构最讲究，按认知科学分了三层——**情景记忆**（会话历史，SQLite + FTS5，
  回答「发生过什么」）、**语义记忆**（偏好与习惯的蒸馏态，回答「你是谁」）、
  **程序记忆**（`~/.hermes/skills/` 下的 Markdown，回答「这事怎么做」），
  另有可选的 Honcho 模块做用户建模。
- **DeepAgents**：不站队，给你可插拔 backend——内存态、本地磁盘、LangGraph store，
  或者按路径组合路由，带读写权限规则。

值得注意的是收敛点：**Markdown 文件成了 agent 记忆的通用货币。** 连结构最工程化的
Hermes，程序记忆也是 Markdown；连最面向开发者的 DeepAgents，也把文件系统当一等公民。
原因不难理解——人能读、能审、能 diff、能进 git。这对下文的安全维度非常重要。

## 维度四：技能与自我改进

`SKILL.md` 正在成为跨 harness 的事实标准（[agentskills.io](https://agentskills.io) 开放标准），
四家都支持，但「技能从哪来」分成了两派：

**人写/社区装：** Claude Code 的技能靠手工编写或生态安装；OpenClaw 有 ClawHub，
社区技能数以万计，装技能像装 npm 包。

**Agent 自己长：** Hermes 的招牌是**五段学习环**，每次任务完成后自动跑：
筛选记忆 → 提炼新技能 → 修正误触发的旧技能 → FTS5 建立可检索索引 → 更新用户模型。
官方手册的说法是：「到第十次时，它已经知道你偏好 `httpx` 而不是 `requests`」——
不用教，它自己总结。

自我改进听起来很美，但 Hermes 自己的文档也承认其中的张力：

> 如果你打算每天 review 它的自我改进，那和手工维护 Skills 有什么区别？

Hermes 的默认答案是「改进时不在环上，产出时在环上」（out of the loop for improvements,
on the loop for outputs）。对反馈信号清晰的场景（代码能跑、任务能验收）这是个合理赌注；
对你自己都判断不了对错的模糊领域，自我改进可能把系统性错误越滚越大。

## 维度五：子代理编排

- **Claude Code**：Task 工具派生子代理，各自拿干净上下文，跑完交回一份报告。
  子代理类型可定制（探索、规划、审查……）。
- **DeepAgents**：内置 `task` 工具，机制几乎同构——**fresh context + 自主执行 + 单次交付**，
  子代理不与主代理来回聊天，只交最终结果。这是控制上下文污染的关键设计。
- **Hermes**：`delegate_task` 派生子代理，但有个刻意的设计约束：**并发硬上限 3 个**。
  在「自治 agent 可能失控膨胀」这件事上，Hermes 选择了物理限速。
- **OpenClaw**：编排相对弱，重心在渠道与主动性而非并行分解。多设备「节点」
  （macOS/iOS/Android）连接网关提供能力，更像分布式的手和眼，而不是分布式的脑。

## 维度六：权限、沙箱与安全边界

这个维度上四家的姿态差异，直接反映了各自的用户画像。

- **Claude Code**：权限模式（每次确认 / 自动接受编辑 / 计划模式）+ hooks 拦截 +
  沙箱化的命令执行。默认假设你在重要的代码库上工作，宁可多问。
- **Hermes**：40+ 内置工具全部走**显式白名单**，沙箱执行，工具集 opt-in。
  记忆是可检查的 SQLite，技能是可 diff 的 Markdown——「技术上可回滚」是明确设计目标。
- **DeepAgents**：human-in-the-loop 中断点、文件系统 backend 的读写权限规则、可选沙箱。
  把安全策略的决定权交给开发者。
- **OpenClaw**：网关层有配对信任模型（新设备需审批、签发 device token、仅本机回环可自动通过），
  但作为拿到你 WhatsApp、邮箱、日历的个人助理，它的攻击面天然巨大。2026 年已有安全研究者
  [以 OpenClaw 为案例系统分析自治 agent 的威胁面](https://arxiv.org/pdf/2603.12644)。
  「文件即 agent」的透明性在这里帮了大忙——至少你能看清它被改成了什么样。

一个值得记住的原则：**agent 的能力半径应该和你的审计能力半径匹配。**
四家里透明度最高的方案（纯文件）恰恰来自能力半径最大的产品（个人助理），这不是巧合。

## 维度七：模型耦合，还是模型无关

最后一条维度关乎战略。

**Claude Code 是紧耦合路线**：harness 参与模型后训练，模型「见过」自己的工具和提示结构。
好处是开箱即用的默契——同样的模型在别的 harness 里往往发挥不出同等水平。

**另外三家全是模型无关路线**：Hermes 支持 OpenRouter 的 200+ 模型、直连 API、
本地 Ollama（隐私优先方案），DeepAgents 接任何 LangChain 支持的模型，OpenClaw 同样可换后端。
代价是拿不到「联合训练」的红利，需要用 harness 工程弥补。

而 harness 工程确实能弥补相当多——LangChain 给出的证据是：**deepagents 在 Terminal Bench 2.0
上从 Top 30 冲进 Top 5，只改了 harness，没换模型。** 这大概是「harness 独立价值」最硬的一个数据点。

顺带一提，模型无关路线也有生态风险：据 Hermes 社区文档记载，2026 年 4 月起 Anthropic
限制了第三方工具使用 Claude 订阅账号，第三方 harness 只能走按量付费 API——
耦合与开放之争，商业层面已经先打起来了。

## 工程实践：怎么选，怎么落地

如果你是在选一个用：

- **主业是写代码**，要模型与工具的最深协同 → **Claude Code**。
- **想要一个活在聊天软件里的助理**，愿意自己托管、看重可审计 → **OpenClaw**。
- **想要常驻后台、越用越懂你的自治 agent**，且你的场景有清晰的对错反馈 → **Hermes**。
- **在给自己的产品造 agent**，需要掌控循环、状态和部署 → **DeepAgents**。

如果你是在自己设计 harness，四家的公约数就是现成的检查清单：

1. **先定时钟**：人、消息、定时器还是状态机驱动？这决定后面所有设计。
2. **上下文当稀缺资源管**：落盘 + 渐进披露 + 子代理隔离，三板斧先备齐。
3. **记忆用 Markdown 打底**：能读、能 diff、能审计，别急着上数据库。
4. **子代理只交报告不闲聊**：fresh context + 单次交付是防上下文污染的最佳实践。
5. **权限白名单起步**，能力半径跟着审计能力走。
6. **别迷信自我改进**：没有清晰反馈信号的领域，自动学习环是错误放大器。

## 收敛与分歧

把四个 harness 摆在一起，收敛的部分多得惊人：Markdown 指令文件、文件系统作外部记忆、
MCP 接工具、SKILL.md 渐进披露、子代理隔离上下文——殊途同归，这些大概率就是 agent
基础设施的「标准件」了。

真正的分歧只剩一个，而且是哲学层面的。Steinberger 在访谈里反复讲：早期 harness 复杂，
是因为模型弱、需要搀扶；**模型每强一代，harness 就该薄一层**——他的终局里 harness
趋近于消失。LangChain 的赌注恰恰相反：模型会继续换代，但**harness 是沉淀工程价值的持久层**，
Terminal Bench 上那个 Top 30 → Top 5 就是宣言。

一边说脚手架终将拆掉，一边说脚手架就是建筑本身。未来两年 agent 基础设施最重要的走向，
大概就藏在这两句话的胜负里。

---

## 参考链接

- [The Anatomy of an Agent Harness — LangChain Blog](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)
- [langchain-ai/deepagents — GitHub](https://github.com/langchain-ai/deepagents) / [Deep Agents overview — LangChain Docs](https://docs.langchain.com/oss/python/deepagents/overview)
- [Gateway architecture — OpenClaw Docs](https://docs.openclaw.ai/concepts/architecture)
- [OpenClaw Explained: Architecture & Alternatives — Turing Post](https://www.turingpost.com/p/openclaw)
- [Inside OpenClaw: How a Persistent AI Agent Actually Works — DEV](https://dev.to/entelligenceai/inside-openclaw-how-a-persistent-ai-agent-actually-works-1mnk)
- [Hermes Agent v0.9 Review: Nous Research Setup, Best Models, Harness](https://www.heyuan110.com/posts/ai/2026-04-14-hermes-agent-guide/)
- [Who is OpenClaw creator Peter Steinberger? — Fortune](https://fortune.com/2026/02/19/openclaw-who-is-peter-steinberger-openai-sam-altman-anthropic-moltbook/)
- [Uncovering Security Threats in Autonomous Agents: A Case Study of OpenClaw — arXiv](https://arxiv.org/pdf/2603.12644)
- [Claude Code 官方文档](https://code.claude.com/docs)
