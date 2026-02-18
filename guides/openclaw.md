# OpenClaw Implementation Guide

OpenClaw has built-in support for spawning sub-agents via `sessions_spawn`. This guide shows how to implement the LLM Microservices pattern using OpenClaw's native tools.

## Prerequisites

- OpenClaw installed and running
- Gateway configured with your LLM provider
- Understanding of your main session vs isolated sessions

## Core Tool: `sessions_spawn`

The `sessions_spawn` tool creates an isolated sub-agent session with its own context.

```python
sessions_spawn(
    task="Your focused task prompt here",
    runTimeoutSeconds=180,  # 3 minutes max
    agentId="optional-custom-agent",  # uses default if omitted
    model="sonnet",  # or "default" for cheaper models
    label="Short label for tracking"  # optional
)
```

## Example: Dev Agent for Code Fix

```python
# Spawn a dev-agent to fix a specific bug
result = sessions_spawn(
    task="""
You are a dev agent. Make a focused code change.

## Context
File: /app/src/risk_manager.py
Bug: `strategy.entry_price` doesn't exist — should be `strategy.entry`

## Task
1. Find: grep -n "entry_price" /app/src/risk_manager.py
2. Replace strategy.entry_price → strategy.entry
3. Verify: python3 -c "from src.risk_manager import X; print('OK')"

## Constraints
- Only change this one attribute reference
- Do NOT restart services
- Do NOT modify unrelated code

## Return
- File and line changed
- Verification output
""",
    runTimeoutSeconds=120,
    model="sonnet"  # code changes need reasoning
)
```

The sub-agent runs in isolation and auto-announces its result back to the main session when done.

## Example: Research Agent for Investigation

```python
result = sessions_spawn(
    task="""
You are a research agent. Investigate and report.

## Symptom
DOT/USDT strategies failing with "price outside range $2.0-$100.0"

## Investigation Steps
1. grep -n "2\.0.*100\|price_ranges" /app/src/validators.py
2. Check current DOT price via CCXT
3. Identify the mismatch

## Return
- File + line number of hardcoded range
- Current actual price
- Root cause (one sentence)
- Recommended fix (a/b/c options)
""",
    runTimeoutSeconds=90,
    model="default"  # research doesn't need expensive models
)
```

## Example: Analysis Agent for Data Fetch

```python
result = sessions_spawn(
    task="""
You are an analysis agent. Fetch and structure data.

## Task
Run this Python script:
```python
import ccxt
e = ccxt.bybit({'apiKey': 'KEY', 'secret': 'SECRET', 'options': {'defaultType': 'swap'}})
positions = [p for p in e.fetch_positions() if float(p.get('contracts') or 0) > 0]
for p in positions:
    print(f"{p['symbol']}: pnl={p.get('unrealizedPnl')}")
```

## Return
JSON: {"positions": [...], "total_pnl": N}
""",
    runTimeoutSeconds=60,
    model="default"
)
```

## Monitoring Sub-Agents

Check active and recent sub-agents:

```python
subagents(action="list")
```

Returns:
```json
{
  "active": [...],
  "recent": [
    {"label": "Fix entry_price bug", "status": "done", "tokens": 1107},
    {"label": "Investigate DOT error", "status": "done", "tokens": 9635}
  ]
}
```

## Steering or Killing

If a sub-agent is running too long or off track:

```python
# Steer it back on course
subagents(
    action="steer",
    target="agent:main:subagent:abc123",
    message="Focus only on the specific file mentioned. Don't explore the codebase."
)

# Or kill it entirely
subagents(action="kill", target="agent:main:subagent:abc123")
```

## Retrieving Results

When a sub-agent completes, OpenClaw auto-announces the result to the main session. To fetch history programmatically:

```python
sessions_history(
    sessionKey="agent:main:subagent:abc123",
    limit=5
)
```

## Best Practices for OpenClaw

### 1. Keep Task Prompts Self-Contained
Sub-agents start fresh. Include all context they need in the task prompt — don't assume they can read your MEMORY.md or conversation history.

### 2. Use the Right Model
- `model="sonnet"` for code changes, complex reasoning
- `model="default"` for research, analysis, simple tasks
- The orchestrator can use a cheaper model since it mostly coordinates

### 3. Set Appropriate Timeouts
- Simple analysis: 60s
- Research: 90–120s
- Code changes: 120–180s
- Complex features: 300s (5 min max)

### 4. Don't Spawn Recursively
Sub-agents spawning sub-agents leads to coordination chaos. Keep it flat: orchestrator → workers.

### 5. Orchestrator Owns Side Effects
Workers should report findings and make file changes. The orchestrator:
- Restarts services
- Sends user notifications
- Commits to git
- Deploys changes

### 6. Review Before Applying
Always read the sub-agent's output before restarting services or merging changes. The pattern is about efficiency, not blind automation.

## Complete Example Session

```python
# Orchestrator wants to fix two bugs in parallel

# Spawn two dev-agents simultaneously
sessions_spawn(
    task="Fix entry_price → entry in risk_manager.py (details...)",
    runTimeoutSeconds=120
)

sessions_spawn(
    task="Update DOT price range in validators.py (details...)",
    runTimeoutSeconds=90
)

# Both run in parallel, main session waits
# When done, orchestrator reviews and restarts once:
# exec("sudo systemctl restart myservice")
```

## Token Savings in Practice

With OpenClaw, a typical session:

| Metric | Traditional | With Sub-Agents |
|--------|-------------|-----------------|
| Main session context | 50k+ tokens | 5–10k tokens |
| Worker tasks (4x) | 4 × 50k = 200k | 4 × 1k = 4k |
| **Total** | 250k tokens | 14k tokens |
| **Savings** | — | **94%** |

The savings compound as the session runs longer.

---

*This pattern is actively used in production with OpenClaw managing live trading systems.*
