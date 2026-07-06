---
title: "Claude Code vs OpenClaw vs Hermes vs DeepAgents: A Deep Dive into Harness Design"
date: 2026-07-07T01:15:00+08:00
draft: false
tags: ["Agent Harness"]
categories: ["System Design"]
description: "Comparing four representative agent harnesses across the core loop, context engineering, memory, skills and self-improvement, subagents, security boundaries, and model coupling — plus practical guidance on choosing one."
---

Over the past year, "agent" has become one of the most overused words in software. But what
actually determines whether an agent is good day-to-day is usually not the model — it's the
layer wrapped around it: the **harness**. LangChain's
[The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)
offers a clean definition:

> Agent = Model + Harness. The harness is "every piece of code, configuration, and execution
> logic that isn't the model itself."

The model sets the ceiling; the harness sets the floor you actually touch every day — how
context is managed, where memory lives, how tools are exposed, how permissions are scoped,
and who drives the loop. This post takes apart four representative harnesses and looks at
the different answers they give to the same set of questions.

## The Four Contenders

| | Positioning | Maker | Loop driver | Memory substrate |
|---|---|---|---|---|
| **Claude Code** | Interactive coding agent | Anthropic | Human-driven terminal session | CLAUDE.md / AGENTS.md + memory directory |
| **OpenClaw** | Self-hosted personal assistant | Peter Steinberger (community) | Message gateway + heartbeat timer | Markdown files on disk |
| **Hermes** | Always-on autonomous agent | Nous Research | Multi-channel gateway + cron | Three-layer memory (SQLite + Markdown) |
| **DeepAgents** | Harness SDK for developers | LangChain | Developer-defined graph runtime | Pluggable virtual filesystem |

Quick background:

- **Claude Code** is Anthropic's terminal coding agent and the flagship of the
  "harness co-evolves with the model" school — LangChain's post specifically notes that
  Claude Code and Codex undergo **post-training with their harnesses integrated**.
- **OpenClaw** is the self-hosted personal assistant Peter Steinberger released in late 2025
  (born as Clawdbot, briefly Moltbot, settled as OpenClaw). Within months it became one of
  the fastest-growing open-source agent projects; Steinberger himself
  [joined OpenAI in February 2026](https://fortune.com/2026/02/19/openclaw-who-is-peter-steinberger-openai-sam-altman-anthropic-moltbook/).
- **Hermes** is Nous Research's open-source agent that productizes the very idea of
  "Harness Engineering," built around self-improvement and 24/7 background operation.
- **DeepAgents** is LangChain's batteries-included harness
  ([langchain-ai/deepagents](https://github.com/langchain-ai/deepagents)) — not an end-user
  product but a foundation for developers building their own agents on the LangGraph runtime.

They target completely different scenarios, yet answer the same design questions. Let's go
dimension by dimension.

## Dimension 1: Who Drives the Loop

This is where the four diverge the most.

**Claude Code: a human holds the wheel.** At its core is a REPL-style session — you say
something, it runs a round (reads code, edits files, runs tests), and comes back to you.
Plan mode, permission prompts, and hooks are all designed around "a human is always on the
loop." It can run background subtasks, but the human sets the session's rhythm.

**OpenClaw: gateway plus heartbeat.** A long-lived Gateway daemon (listening on
`127.0.0.1:18789` by default) owns all messaging surfaces over WebSocket — WhatsApp,
Telegram, Slack, Discord, Signal, iMessage. Messages come in and get routed to the agent
loop. The more interesting part is the **heartbeat**: every 30 minutes by default, the agent
is woken up to read the task checklist in `HEARTBEAT.md` and see what it should proactively
do. The agent keeps turning even when nobody is around.

**Hermes: resident by default.** Also a multi-channel gateway (Telegram, Discord, Slack,
WhatsApp, Signal, iMessage, WeChat, CLI — 12+ entry points), but the design center of
gravity is 24/7 autonomous operation — one officially recommended deployment is a $5 VPS
with a sub-500MB memory footprint. With cron scheduling plus a learning loop that fires
automatically after each task (more below), it goes one step further than OpenClaw: not just
"keeps running while you're away," but "keeps modifying itself while it runs."

**DeepAgents: the loop is a graph you write.** As an SDK it doesn't presuppose a driver —
developers define the graph with LangGraph, and the runtime provides durable execution,
streaming, and human interrupt points. The shape of the loop is itself part of your product
design.

In engineering terms, this dimension boils down to one question: **what is your agent's
"clock"?** The human's next message (Claude Code), external messages (the OpenClaw/Hermes
gateways), a timer (heartbeat/cron), or a graph runtime's state machine (DeepAgents).
Decide the clock first; everything else follows.

## Dimension 2: Context Engineering

All four face the same enemy: context rot — model performance degrading as the window fills
up. The solutions have converged remarkably; the details differ.

- **Claude Code**: automatic compaction (summarize when the window nears capacity), tool
  output offloaded to disk, subagents with isolated contexts, and **progressive disclosure**
  for Skills — only names and descriptions go into the system prompt; the full text is read
  on demand.
- **OpenClaw**: a textbook case of the same idea — context assembly injects only each
  skill's name, description, and file path, and the model **reads `SKILL.md` on demand**
  once it judges a skill relevant. Tens of thousands of community skills won't blow up the
  context.
- **Hermes**: pushes retrieve-on-demand into the memory layer — new sessions don't load
  history; instead, SQLite FTS5 full-text search pulls only the relevant fragments. After
  months of use, startup latency doesn't grow with history.
- **DeepAgents**: a virtual filesystem as the offload target (intermediate results go to
  files, not context), plus built-in summarization and prompt caching.

The consensus is settled: **context is a scarce resource — offload everything you can,
lazy-load everything you can.** The differences are only in where things land (files /
SQLite / pluggable backends) and who decides what gets loaded (the model reads vs. a
retriever pushes).

## Dimension 3: Where Memory Lives

- **Claude Code**: `CLAUDE.md` / `AGENTS.md` carry project conventions, plus an
  automatically maintained memory directory. The philosophy: memory is documentation written
  for the model.
- **OpenClaw**: the most extreme and the most charming — **the agent is a pile of files on
  disk**. `SOUL.md` defines the personality, `MEMORY.md` holds long-term memory,
  `HEARTBEAT.md` is the task list. Want to change the agent's behavior? Edit a file. Want to
  audit what it knows? Open the folder. No database, no hidden state.
- **Hermes**: the most structured, borrowing three memory types from cognitive science —
  **episodic** (session history, SQLite + FTS5, answers "what happened"), **semantic**
  (distilled preferences and habits, answers "who you are"), and **procedural** (Markdown
  under `~/.hermes/skills/`, answers "how to do X"), with an optional Honcho module for user
  modeling.
- **DeepAgents**: refuses to pick a side — pluggable backends: in-memory state, local disk,
  LangGraph store, or composite routing by path, with read/write permission rules.

Note the convergence point: **Markdown files have become the common currency of agent
memory.** Even Hermes, the most engineered of the four, keeps procedural memory in Markdown;
even DeepAgents, the most developer-oriented, treats the filesystem as a first-class
citizen. The reason is simple — humans can read it, review it, diff it, and put it in git.
That matters enormously for the security dimension below.

## Dimension 4: Skills and Self-Improvement

`SKILL.md` is becoming a de facto cross-harness standard (the
[agentskills.io](https://agentskills.io) open spec). All four support it, but "where skills
come from" splits into two camps:

**Written by humans / installed from a community:** Claude Code skills are hand-written or
installed from the ecosystem; OpenClaw has ClawHub with tens of thousands of community
skills — installing a skill feels like installing an npm package.

**Grown by the agent itself:** Hermes's signature is its **five-stage learning loop**,
which runs automatically after each task: curate memory → extract new skills → refine skills
that misfired → index everything for FTS5 recall → update the user model. As the official
handbook puts it: "By the tenth time, it knows you prefer `httpx` over `requests`" — no
teaching required.

Self-improvement sounds great, but Hermes's own docs admit the tension:

> If you are going to review its self-improvements daily, what is the difference from
> manually maintaining your Skills?

Hermes's default answer is "out of the loop for improvements, on the loop for outputs."
For domains with clear feedback signals (code compiles, tasks are verifiable) that's a
reasonable bet; for fuzzy domains where you can't judge correctness yourself, a
self-improvement loop can compound systematic errors.

## Dimension 5: Subagent Orchestration

- **Claude Code**: the Task tool spawns subagents, each with a clean context, returning a
  single report when done. Subagent types are customizable (exploration, planning,
  review...).
- **DeepAgents**: a built-in `task` tool with an almost isomorphic design — **fresh context
  + autonomous execution + single handoff**. Subagents don't chat back and forth with the
  main agent; they deliver one final result. This is the key design for containing context
  pollution.
- **Hermes**: `delegate_task` spawns subagents, with one deliberate constraint: a **hard cap
  of 3 concurrent subagents**. On the question of "autonomous agents ballooning out of
  control," Hermes chose a physical speed limit.
- **OpenClaw**: orchestration is comparatively weak; the focus is on channels and
  proactivity rather than parallel decomposition. Multi-device "nodes" (macOS/iOS/Android)
  connect to the gateway to contribute capabilities — more like distributed hands and eyes
  than a distributed brain.

## Dimension 6: Permissions, Sandboxing, and the Security Boundary

Each project's posture on this dimension directly reflects its user profile.

- **Claude Code**: permission modes (confirm each action / auto-accept edits / plan mode) +
  hook interception + sandboxed command execution. The default assumption is that you're
  working on a codebase that matters — better to ask too often.
- **Hermes**: 40+ built-in tools all behind **explicit allowlists**, sandboxed execution,
  opt-in toolsets. Memory is inspectable SQLite, skills are diffable Markdown — "technically
  reversible" is an explicit design goal.
- **DeepAgents**: human-in-the-loop interrupts, read/write permission rules on filesystem
  backends, optional sandboxes. Security policy decisions are handed to the developer.
- **OpenClaw**: the gateway layer has a pairing-based trust model (new devices require
  approval, device tokens are issued, only local loopback can be auto-approved). But as a
  personal assistant holding your WhatsApp, email, and calendar, its attack surface is
  inherently huge. Security researchers have already used OpenClaw as
  [a case study for analyzing autonomous-agent threat surfaces](https://arxiv.org/pdf/2603.12644).
  The "agent as files" transparency helps a lot here — at least you can see exactly what it
  has been changed into.

One principle worth remembering: **an agent's radius of capability should match your radius
of audit.** It's no coincidence that the most transparent design of the four (plain files)
comes from the product with the largest capability radius (a personal assistant).

## Dimension 7: Model-Coupled or Model-Agnostic

The last dimension is strategic.

**Claude Code takes the tightly coupled route**: the harness participates in model
post-training, so the model has "seen" its own tools and prompt structure. The payoff is
out-of-the-box synergy — the same model in another harness often can't perform at the same
level.

**The other three are all model-agnostic**: Hermes supports 200+ models via OpenRouter,
direct APIs, and local Ollama (the privacy-first option); DeepAgents plugs into anything
LangChain supports; OpenClaw can swap backends too. The cost is forgoing the co-training
dividend and making it up with harness engineering.

And harness engineering can make up quite a lot — LangChain's evidence: **deepagents went
from Top 30 to Top 5 on Terminal Bench 2.0 through harness-only changes, same model.**
That's probably the hardest data point yet for the independent value of the harness.

One aside: the model-agnostic route carries ecosystem risk. According to Hermes community
docs, starting April 2026 Anthropic restricted third-party tools from using Claude
subscription accounts, forcing third-party harnesses onto pay-as-you-go APIs — the
coupled-vs-open battle has already begun at the commercial layer.

## The Practical Take: Choosing and Building

If you're choosing one to use:

- **Your main job is writing code** and you want the deepest model-tool synergy →
  **Claude Code**.
- **You want an assistant that lives in your messaging apps**, self-hosted and auditable →
  **OpenClaw**.
- **You want an always-on agent that knows you better over time**, and your domain has
  clear right/wrong feedback → **Hermes**.
- **You're building an agent into your own product** and need control over the loop, state,
  and deployment → **DeepAgents**.

If you're designing your own harness, the common denominators of these four are a
ready-made checklist:

1. **Pick the clock first**: human, message, timer, or state machine? This decides
   everything downstream.
2. **Treat context as a scarce resource**: offloading + progressive disclosure + subagent
   isolation — get all three in place early.
3. **Base memory on Markdown**: readable, diffable, auditable. Don't reach for a database
   too soon.
4. **Subagents deliver reports, not conversations**: fresh context + single handoff is the
   best practice against context pollution.
5. **Start with permission allowlists**, and let the capability radius follow your audit
   radius.
6. **Don't romanticize self-improvement**: in domains without clear feedback signals, an
   automatic learning loop is an error amplifier.

## Convergence and Divergence

Put the four harnesses side by side and the convergence is striking: Markdown instruction
files, the filesystem as external memory, MCP for tools, progressive disclosure via
SKILL.md, context-isolated subagents — different roads, same destination. These are most
likely the "standard parts" of agent infrastructure from here on.

Only one real divergence remains, and it's philosophical. Steinberger has said repeatedly
in interviews that early harnesses were complex because models were weak and needed
scaffolding; **every model generation should make the harness thinner** — in his endgame the
harness approaches disappearance. LangChain's bet is the opposite: models will keep churning,
but **the harness is the durable layer where engineering value accumulates** — that
Top 30 → Top 5 jump on Terminal Bench is their manifesto.

One side says the scaffolding will eventually come down; the other says the scaffolding is
the building. The most important direction in agent infrastructure over the next two years
probably hides in which of those two sentences wins.

---

## References

- [The Anatomy of an Agent Harness — LangChain Blog](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)
- [langchain-ai/deepagents — GitHub](https://github.com/langchain-ai/deepagents) / [Deep Agents overview — LangChain Docs](https://docs.langchain.com/oss/python/deepagents/overview)
- [Gateway architecture — OpenClaw Docs](https://docs.openclaw.ai/concepts/architecture)
- [OpenClaw Explained: Architecture & Alternatives — Turing Post](https://www.turingpost.com/p/openclaw)
- [Inside OpenClaw: How a Persistent AI Agent Actually Works — DEV](https://dev.to/entelligenceai/inside-openclaw-how-a-persistent-ai-agent-actually-works-1mnk)
- [Hermes Agent v0.9 Review: Nous Research Setup, Best Models, Harness](https://www.heyuan110.com/posts/ai/2026-04-14-hermes-agent-guide/)
- [Who is OpenClaw creator Peter Steinberger? — Fortune](https://fortune.com/2026/02/19/openclaw-who-is-peter-steinberger-openai-sam-altman-anthropic-moltbook/)
- [Uncovering Security Threats in Autonomous Agents: A Case Study of OpenClaw — arXiv](https://arxiv.org/pdf/2603.12644)
- [Claude Code official docs](https://code.claude.com/docs)
