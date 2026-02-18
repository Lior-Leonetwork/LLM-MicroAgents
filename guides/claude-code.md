# Claude Code Implementation Guide

Claude Code (Anthropic's CLI agent) supports sub-agent spawning and can implement the LLM Microservices pattern with a few key configurations.

## Prerequisites

- Claude Code CLI installed (`npm install -g @anthropic-ai/claude-code`)
- Anthropic API key configured
- Project with files you want to work on

## Core Pattern: Background Tasks

Claude Code can run tasks in the background. Use this to create isolated workers:

```bash
# In a separate terminal or via shell tool:
claude "You are a dev agent. Fix the bug in risk_manager.py where entry_price should be entry. Only change that one attribute. Report back what you changed." &
```

The `&` runs it in the background. The main Claude Code session continues while the worker runs.

## Better Approach: Shell-Based Worker Spawning

For true isolation, spawn Claude Code instances programmatically:

```python
import subprocess
import json

def spawn_dev_agent(task, project_path, timeout=120):
    """Spawn an isolated Claude Code worker for a focused task."""
    result = subprocess.run(
        ["claude", "--non-interactive", "--max-tokens", "2000", task],
        cwd=project_path,
        capture_output=True,
        text=True,
        timeout=timeout
    )
    return result.stdout

# Usage
output = spawn_dev_agent(
    task="""
    Fix the bug in src/risk_manager.py:
    - Replace strategy.entry_price with strategy.entry
    - Only change this specific attribute
    - Run: python3 -c "from src.risk_manager import X; print('OK')"
    - Report: file, line number, verification result
    """,
    project_path="/path/to/your/project",
    timeout=120
)
print(output)
```

## Parallel Workers

Run multiple Claude Code instances in parallel:

```python
import subprocess
from concurrent.futures import ThreadPoolExecutor, as_completed

tasks = [
    "Fix entry_price bug in risk_manager.py (details...)",
    "Update DOT price range in validators.py (details...)",
    "Add log rotation config for /var/log/app.log (details...)"
]

def run_worker(task):
    return subprocess.run(
        ["claude", "--non-interactive", task],
        capture_output=True,
        text=True,
        timeout=120
    ).stdout

with ThreadPoolExecutor(max_workers=3) as executor:
    futures = {executor.submit(run_worker, task): task for task in tasks}
    for future in as_completed(futures):
        task = futures[future]
        result = future.result()
        print(f"Task: {task[:50]}...\nResult: {result}\n")
```

## Worker Types for Claude Code

### dev-agent
```python
task = """
You are a dev agent. Make ONE focused code change.

FILE: src/validators.py
CHANGE: Update line 43: DOT/USDT range from (2.0, 100.0) to (0.5, 100.0)

CONSTRAINTS:
- Only change this one line
- Do not restart any services
- Do not modify other symbols

VERIFY: python3 -c "from validators import validate; print('OK')"

RETURN: Old line, new line, verification output
"""
```

### research-agent
```python
task = """
You are a research agent. Investigate and report. Do NOT make changes.

SYMPTOM: DOT/USDT failing with "price outside range $2.0-$100.0"

STEPS:
1. grep -n "2\.0.*100" src/validators.py
2. Read the relevant section
3. Check current DOT price: python3 -c "import ccxt; print(ccxt.bybit().fetch_ticker('DOT/USDT:USDT')['last'])"

RETURN:
- File + line number
- Current actual price
- Root cause (one sentence)
- Recommended fix option (a/b/c)
"""
```

### analysis-agent
```python
task = """
You are an analysis agent. Fetch data and return JSON.

RUN:
import ccxt
import json
e = ccxt.bybit({'options': {'defaultType': 'swap'}})
positions = [p for p in e.fetch_positions() if float(p.get('contracts') or 0) > 0]
result = {"positions": [{"symbol": p['symbol'], "pnl": p.get('unrealizedPnl')} for p in positions]}
print(json.dumps(result))

RETURN: Only the JSON output, no additional commentary
"""
```

## Orchestrator Pattern

The orchestrator is your main Claude Code session. It should:

1. **Talk to the user** — interpret requests, clarify intent
2. **Spawn workers** — delegate focused tasks to background instances
3. **Collect results** — read worker output
4. **Apply changes** — restart services, commit, deploy
5. **Stay thin** — don't do the work yourself if a worker can do it

### Example Orchestrator Flow

```
User: "Fix the DOT price validation bug"

Orchestrator:
1. [spawns research-agent] → "Investigate DOT validation failure"
2. [waits] → receives: "Root cause: hardcoded range $2.0-$100.0, actual price $1.35"
3. [spawns dev-agent] → "Update line 43 in validators.py to range (0.5, 100.0)"
4. [waits] → receives: "Changed. Verification passed."
5. [applies] → restarts service
6. [responds] → "Fixed. DOT now accepts prices down to $0.50."
```

## Key Claude Code Flags

| Flag | Purpose |
|------|---------|
| `--non-interactive` | Run without prompts (required for workers) |
| `--max-tokens N` | Limit output length |
| `--model sonnet` | Use Sonnet for reasoning-heavy tasks |
| `--model haiku` | Use Haiku for cheap, fast tasks |
| `--output-format json` | Structured output for programmatic parsing |

## Token Budget Management

Claude Code tracks context. To keep the orchestrator thin:

1. **Don't read unnecessary files** — if a worker can find it, let them
2. **Don't dump large outputs** — summarize, don't paste 200-line diffs
3. **Use memory files sparingly** — workers don't need your MEMORY.md
4. **Clear context periodically** — start fresh sessions for new problem domains

## Example: Parallel Investigation + Fix

```python
# orchestrator.py
import subprocess
import json
from concurrent.futures import ThreadPoolExecutor

def spawn_worker(task, model="haiku", timeout=90):
    """Spawn an isolated Claude Code worker."""
    cmd = [
        "claude",
        "--non-interactive",
        "--max-tokens", "2000",
        "--model", model,
        task
    ]
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=timeout)
    return result.stdout

# Run research and analysis in parallel
with ThreadPoolExecutor(max_workers=2) as executor:
    research_future = executor.submit(
        spawn_worker,
        "Investigate why DOT/USDT fails validation in validators.py (details...)"
    )
    analysis_future = executor.submit(
        spawn_worker,
        "Fetch current DOT price from Bybit and return as JSON (details...)"
    )

    research_result = research_future.result()
    analysis_result = analysis_future.result()

# Orchestrator reviews, then spawns fix worker
print("Research:", research_result)
print("Analysis:", analysis_result)

fix_result = spawn_worker(
    "Update DOT range in validators.py line 43 to (0.5, 100.0) (details...)",
    model="sonnet",  # code change needs reasoning
    timeout=120
)
print("Fix applied:", fix_result)
```

## Best Practices

### 1. One Task Per Worker
Don't give a worker a laundry list. One file, one change, one investigation.

### 2. Explicit Output Format
Tell the worker exactly what to return:
```
RETURN: JSON with keys "file", "line", "old", "new", "status"
```

### 3. Verify in the Worker
Have the worker run syntax checks or tests. Don't let broken code reach the orchestrator.

### 4. Cheap Models for Simple Work
- Haiku for grep, read, summarize
- Sonnet for write, reason, debug

### 5. Timeouts Are Safety Nets
Set them. Workers can get stuck in rabbit holes.

### 6. Orchestrator Owns Side Effects
Workers change files. Orchestrator restarts services, commits, notifies users.

## Limitations

- Claude Code doesn't have native `sessions_spawn` like OpenClaw — you simulate it with subprocess
- Context isn't automatically shared between instances — which is the point
- No built-in sub-agent monitoring — track them yourself with futures/processes

---

*This approach works best when you wrap Claude Code in a thin orchestrator script that handles spawning, collecting, and applying.*
