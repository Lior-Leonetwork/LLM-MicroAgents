# 10,000 Tasks: What We Learned at Scale

In February 2026, our OpenClaw-based multi-agent system crossed **10,015 task completions** — all using the MicroAgents pattern.

Here's what the data shows after running this across a production AI trading system and a VR game development project, continuously, for two months.

---

## System Overview

| | |
|-|-|
| **Deployment** | 6-agent OpenClaw team (3 trading + 3 VR) |
| **Projects** | Crypto trading bot + VR AI companion game |
| **Period** | ~2 months continuous operation |
| **Milestone** | 10,015 completions — 2026-02-21 07:15 UTC |
| **Peak throughput** | 128 tasks in a single 3-hour session |
| **Peak parallel** | 8 simultaneous sub-agents |

---

## Token Savings: Confirmed at Scale

### Individual Task Examples

| Task | Sub-agent tokens | Main session equiv. | Savings |
|------|-----------------|---------------------|---------|
| Directory / structure analysis | 344 | ~15,000 | **97.7%** |
| Config file generation | 585 | ~14,800 | **96.0%** |
| Single-file bug fix | 1,107 | ~13,800 | **92.0%** |
| Root cause investigation (multi-file) | 9,635 | ~39,500 | **75.6%** |

### Averages Across 10k Tasks

| Metric | Value |
|--------|-------|
| Average sub-agent cost | ~500 tokens |
| Average main-session equivalent | ~15,000–50,000 tokens |
| Average savings | **85–97%** |
| Cumulative savings (4-task session) | **85.9%** (~82k tokens → ~12k) |

---

## Parallelism: The Real Multiplier

Single-agent thinking is sequential. MicroAgents are concurrent.

### Real Session: 5 Features, One Sprint (2026-02-18)

14 sub-agents deployed across a single session. In one parallel sprint, 5 independent dev-agents ran simultaneously:

| Agent | Task | Approx. time |
|-------|------|-------|
| dev-agent-1 | 4H trend bias filter (EMA 200 + ADX > 20) | ~8 min |
| dev-agent-2 | Fair Value Gap (FVG) detection | ~7 min |
| dev-agent-3 | Funding rate monitor | ~6 min |
| dev-agent-4 | FVG integration into entry validation | ~9 min |
| dev-agent-5 | Correlation circuit breaker | ~8 min |

**Wall-clock time:** ~10 min (limited by slowest agent)
**Sequential equivalent:** ~38–47 min
**Speed multiplier: ~4x**

Each agent received only the context it needed. No agent waited on another. The orchestrator reviewed all five outputs, applied changes, and restarted the service once.

---

## What Changes at 10k vs 4 Tasks

The pattern holds at scale, but two things emerge that aren't obvious from small samples.

### 1. Compounding Savings

At 4 tasks, savings are obvious but modest. At 10,000 tasks:

- Main session context grows continuously (conversation history, memory files, tool results)
- A fresh sub-agent always starts at ~0 tokens
- **The gap between sub-agent and main-session cost widens the longer you run**

In long-running sessions (8+ hours), main-session context reaches 50k+ tokens. At that point, even a trivial task costs 14,000+ tokens just for the context. The sub-agent still costs 344.

```
Session age →→→→→→→→→→→→→→→→→→→→→→→→→→
Main session cost per task:  ████████████████████████  growing
Sub-agent cost per task:     ███  flat
```

### 2. Model Selection Becomes Critical

At scale, task-model mismatch is the biggest source of avoidable waste:

| Task | Right model | Wrong model | Cost ratio |
|------|-------------|-------------|------------|
| Directory lookup, log scan | fast/cheap (haiku, glm-flash) | sonnet | 10–50x overspend |
| Root cause, complex reasoning | sonnet | fast model | Fails or misses bugs |
| Config generation, simple writes | haiku | opus | 20x overspend |

The rule: **use the cheapest model that can do the job correctly.** At 10k tasks, this compounds enormously.

---

## Throughput at Scale

128 tasks in 3 hours = **42.6 tasks/hour**.

This isn't from running faster. It's from removing context bottlenecks. When each task carries only what it needs (~500 tokens), the orchestrator dispatches dozens of agents without the main session grinding under accumulated weight.

The agents don't slow down session 500 the way they would in a traditional approach — because each one starts fresh.

---

## Failure Modes (What Breaks the Pattern)

After 10k tasks, the failure modes are clear:

**#1: Vague task scoping**
Workers given ambiguous prompts expand their own context to compensate. A 200-token task definition becomes a 5,000-token context as the agent reads files it wasn't told to skip. The fix: explicit constraints in every prompt (`Only change this one line`, `Do not read unrelated files`).

**#2: Wrong worker type**
A dev-agent used for investigation loads code files unnecessarily. A research-agent used for code changes hedges instead of committing. Match the worker type to the task category.

**#3: Long-running stateful workers**
If a worker needs to maintain state across multiple steps, it's doing orchestrator work. Break it into discrete tasks with the orchestrator managing state between them.

**#4: Skipping output contracts**
Workers that return free-form text instead of structured output (JSON, diff, `✅/❌`) force the orchestrator to parse and interpret instead of act. This adds tokens to the orchestrator and introduces error.

---

## Cost in Practice

Running on OpenRouter with Llama 3.3 70B for analysis tasks and selective Sonnet for dev tasks:

| | Cost |
|-|------|
| Autonomous research budget (OpenRouter) | ~$15/month configured |
| Typical 4-task mixed session | ~$0.10–$2.00 |
| Estimated equivalent single-agent cost | ~$5–$30 |

The pattern paid for itself within the first week of deployment.

---

## Summary

**78% average savings** is what we shipped. Here's what 10,000 tasks confirms:

- The savings are **real and reproducible** across wildly different task types
- **Parallelism compounds the value** — 5 agents in parallel ≈ 4x output at same cost and time
- The pattern **scales linearly** — no degradation at 10k tasks vs 4
- **Model selection** matters more than architecture at scale — wrong model choice costs more than wrong pattern choice

The pattern is simple. The discipline is in the task scoping.

---

*All data from: OpenClaw 6-agent team, production systems, February 2026.*
*Projects: LeoTrader (crypto trading AI) and VR AI companion game.*
