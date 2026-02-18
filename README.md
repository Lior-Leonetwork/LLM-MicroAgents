<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/dce59c25-c270-4a11-83ba-36fe7ef7d52a" />

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/openclaw/openclaw/main/docs/assets/openclaw-logo-text.png">
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/openclaw/openclaw/main/docs/assets/openclaw-logo-text-dark.png">
  <img alt="OpenClaw" src="https://raw.githubusercontent.com/openclaw/openclaw/main/docs/assets/openclaw-logo-text-dark.png" height="60">
</picture>

# LLM MicroAgents

> **Sub-agents as stateless microservices.** Keep the orchestrator thin. Workers get only what they need.

**78% average token savings** • Works with any LLM • Battle-tested in production • Native support in [OpenClaw](https://openclaw.ai)

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
├── dev-agent      → "Fix bug in file X"       (500 tokens)
├── research-agent → "Find root cause"         (300 tokens)
└── analysis-agent → "Fetch data, return JSON" (400 tokens)
```

**Result:** ~78% token savings. Faster responses. Cleaner context.

---

## Why OpenClaw is the Best Fit

OpenClaw has **native, first-class support** for this pattern via its built-in `sessions_spawn` tool — no subprocess wrangling, no shell scripting, no glue code.

```python
# One call. Isolated context. Auto-announces result when done.
sessions_spawn(
    task="Fix entry_price → entry in risk_manager.py (details...)",
    runTimeoutSeconds=120,
    model="sonnet"
)
```

Other platforms require you to simulate isolation with subprocesses and futures. OpenClaw gives you:

| Feature | OpenClaw | Other platforms |
|---------|----------|-----------------|
| Native `sessions_spawn` | ✅ Built-in | ❌ Subprocess workaround |
| Auto result announcement | ✅ Yes | ❌ Manual collection |
| Sub-agent monitoring | ✅ `subagents(action="list")` | ❌ Track manually |
| Steer / kill mid-flight | ✅ `subagents(action="steer")` | ❌ Kill process |
| Multi-channel orchestration | ✅ WhatsApp, Telegram, etc. | ❌ No |
| Parallel spawning | ✅ Native | ⚠️ ThreadPoolExecutor |

See [`guides/openclaw.md`](./guides/openclaw.md) for the full OpenClaw implementation guide.

---

## Real Numbers

| Task | Sub-agent | Main Session | Savings |
|------|-----------|--------------|---------|
| Analyze codebase structure | 344 | ~15,000 | **97.7%** |
| Fix validation bug | 1,400 | ~50,000 | **97.2%** |
| Investigate root cause | 9,600 | ~40,000 | **76.0%** |
| Log rotation setup | 585 | ~14,800 | **96.0%** |

See [`examples/tokens-saved.md`](./examples/tokens-saved.md) for detailed breakdowns.

---

## Core Principles

| Principle | What it means |
|-----------|---------------|
| **Context isolation** | Workers start fresh — no history, no memory |
| **Single responsibility** | One worker, one task |
| **Stateless workers** | No cross-worker state, no accumulation |
| **Structured output** | JSON, diff summaries — no essays |
| **Orchestrator control** | Workers build, orchestrator applies |

---

## When to Use

✅ **Good fit:** Code changes, investigations, data analysis, documentation, parallel tasks

❌ **Not a fit:** Real-time decisions, user-facing chat, stateful workflows

---

## Quick Start

### OpenClaw (recommended)

```python
# Spawn a dev-agent with a focused task
result = sessions_spawn(
    task="""
You are a dev agent. Make a focused code change.

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
""",
    runTimeoutSeconds=120,
    model="sonnet"
)
```

### Claude Code / Other platforms

```python
import subprocess
from concurrent.futures import ThreadPoolExecutor

def spawn_worker(task, model="haiku", timeout=90):
    result = subprocess.run(
        ["claude", "--non-interactive", "--model", model, task],
        capture_output=True, text=True, timeout=timeout
    )
    return result.stdout
```

See [`guides/claude-code.md`](./guides/claude-code.md) for the full subprocess-based approach.

---

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

---

## Worker Types

### dev-agent
**Purpose:** Code changes, bug fixes, feature implementation
**Model:** Sonnet (needs reasoning)
**Timeout:** 2–3 min
**Output:** What changed + verification result

### research-agent
**Purpose:** Investigation, log analysis, root cause finding
**Model:** Fast/cheap (reading + summarizing)
**Timeout:** 1–2 min
**Output:** Findings + recommended fix (does NOT apply changes)

### analysis-agent
**Purpose:** Data fetch, metrics, pattern detection
**Model:** Fast/cheap
**Timeout:** 1 min
**Output:** Structured JSON or markdown table

**Templates:** [`templates/`](./templates/) — ready-to-use prompts for each worker type.

---

## Documentation

| File | What's inside |
|------|---------------|
| [`ARCHITECTURE.md`](./ARCHITECTURE.md) | Orchestrator patterns, worker types, anti-patterns |
| [`guides/openclaw.md`](./guides/openclaw.md) | Native `sessions_spawn` implementation (recommended) |
| [`guides/claude-code.md`](./guides/claude-code.md) | Subprocess-based workers for Claude Code |
| [`examples/tokens-saved.md`](./examples/tokens-saved.md) | Real token savings data |

---

## License

MIT — use it, fork it, ship it.

---

*Built while building multi-agent AI systems. Runs best on [OpenClaw](https://openclaw.ai) 🦞*
