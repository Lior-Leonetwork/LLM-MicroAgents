# MicroAgents

> **Sub-agents as stateless microservices.** Keep the orchestrator thin. Workers get only what they need.

**78% average token savings** • Works with any LLM • Battle-tested in production

---

## The Problem

Traditional LLM agents accumulate context. Every message, file read, and tool result bloats the conversation until you're paying for 50k+ tokens just to fix a bug.

```
Main session: [50k tokens of history] → "fix this typo"
```

**Slow. Expensive. Error-prone.**

## The Solution

Treat sub-agents like microservices. Spawn them with minimal context, let them do one thing, then die.

```
Orchestrator (thin)
├── dev-agent      → "Fix bug in file X"      (500 tokens)
├── research-agent → "Find root cause"        (300 tokens)
└── analysis-agent → "Fetch data, return JSON" (400 tokens)
```

**Result:** ~78% token savings. Faster responses. Cleaner context.

## Real Numbers

| Task | Sub-agent | Main Session | Savings |
|------|-----------|--------------|---------|
| Analyze codebase structure | 344 | ~15,000 | **97.7%** |
| Fix validation bug | 1,400 | ~50,000 | **97.2%** |
| Investigate root cause | 9,600 | ~40,000 | **76.0%** |

See [`examples/tokens-saved.md`](./examples/tokens-saved.md) for detailed breakdowns.

## Core Principles

| Principle | What it means |
|-----------|---------------|
| **Context isolation** | Workers start fresh — no history, no memory |
| **Single responsibility** | One worker, one task |
| **Stateless workers** | No cross-worker state, no accumulation |
| **Structured output** | JSON, diff summaries — no essays |
| **Orchestrator control** | Workers build, orchestrator applies |

## When to Use

✅ **Good fit:** Code changes, investigations, data analysis, documentation, parallel tasks

❌ **Not a fit:** Real-time decisions, user-facing chat, stateful workflows

## Quick Start

```python
# Spawn a dev-agent with a focused task
task = """
## Context
File: /app/service.py
Bug: `entry_price` doesn't exist — should be `entry`

## Task
1. Replace strategy.entry_price → strategy.entry
2. Verify: python3 -c "from service import X; print('OK')"

## Constraints
- Only change this one reference
- Don't restart services

## Return
- Lines changed (file:line)
- Verification result
"""
```

**Templates:** [`templates/`](./templates/) — ready-to-use prompts for dev, research, and analysis agents.

## Documentation

| File | What's inside |
|------|---------------|
| [`ARCHITECTURE.md`](./ARCHITECTURE.md) | Orchestrator patterns, worker types, anti-patterns |
| [`guides/openclaw.md`](./guides/openclaw.md) | Native `sessions_spawn` implementation |
| [`guides/claude-code.md`](./guides/claude-code.md) | Subprocess-based workers for Claude Code |
| [`examples/`](./examples/) | Real token savings data |

## Architecture

```
┌─────────────────────────────────────────────┐
│              ORCHESTRATOR                    │
│         (main session, persists)             │
└────────────────────┬────────────────────────┘
                     │ spawns
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │dev-agent│ │research │ │analysis │
    └────┬────┘ └────┬────┘ └────┬────┘
         │           │           │
         └───────────┴───────────┘
                     ▼
         OUTPUT CONTRACT (JSON)
                     │
                     ▼
         ORCHESTRATOR reviews & applies
```

**Key insight:** Workers start fresh, do one thing, and die. The orchestrator stays thin.

## License

MIT — use it, fork it, ship it.

---

*Built while building multi-agent AI systems.*
