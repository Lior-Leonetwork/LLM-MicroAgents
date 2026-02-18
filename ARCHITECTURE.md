# Architecture

## Roles

### Orchestrator (Main Session)
The orchestrator is the only session that persists. It:
- Talks to the user
- Maintains high-level system state
- Decides **what** needs to be done
- Spawns workers with precise task definitions
- Reviews worker output before applying it
- Handles all external side effects (notifications, service restarts, deploys)

The orchestrator should stay as thin as possible. If it starts doing the work itself, token costs go up and the pattern breaks down.

### Workers (Sub-Agents)
Workers are ephemeral. They:
- Start fresh (no history, no loaded context)
- Receive a self-contained task prompt
- Execute, return structured output, and terminate
- Never communicate with other workers
- Never trigger user-facing actions directly

## Worker Types

### dev-agent
**Purpose:** Code changes, bug fixes, feature implementation  
**Input:** File path + specific change + constraints  
**Output:** What changed (file, line numbers) + syntax/test verification  
**Model:** Sonnet-class (needs reasoning for code)  
**Timeout:** 3–5 min  

### research-agent
**Purpose:** Investigation, log analysis, root cause finding  
**Input:** Symptom description + files/commands to inspect  
**Output:** Findings + recommended fix (does NOT apply changes)  
**Model:** Fast/cheap (mostly reading + summarizing)  
**Timeout:** 2–3 min  

### analysis-agent
**Purpose:** Data analysis, metrics, pattern detection  
**Input:** API credentials + specific question + output format  
**Output:** Structured JSON or markdown table  
**Model:** Fast/cheap (computation, not reasoning)  
**Timeout:** 1–2 min  

## Task Scoping

The quality of a sub-agent's output is directly proportional to the quality of its task definition.

**Good task definition:**
```
## Context
File: /app/src/validators/price_validator.py
Lines 43-47 contain hardcoded price ranges for DOT/USDT (2.0, 100.0).
Current actual price: $1.35

## Task
Update line 43 to use range (0.5, 100.0) instead of (2.0, 100.0).

## Constraints
- Only change this one line
- Do not touch any other symbols
- Do not restart any services

## Return
- The old and new line content
- Output of: python3 -c "from validators.price_validator import validate; print('OK')"
```

**Bad task definition:**
```
Fix the price validation issues in the validator module.
```

## Output Contracts

Workers should return a predictable structure. The orchestrator pattern-matches on output, not on free-form text.

Define the return format explicitly:
- JSON for machine-readable results
- Bullet list for human-readable summaries
- `file:line` format for code locations
- `✅ / ❌` for pass/fail verification

## Parallelism

Multiple workers can run simultaneously on independent tasks:

```
Orchestrator
├── [spawn] research-agent: "Why is DOT failing price validation?"
├── [spawn] dev-agent: "Implement fresh sweep bonus in risk_manager.py"
└── [spawn] analysis-agent: "Get current PnL for all positions"
         ↓ (wait for all)
Orchestrator reviews results, applies changes, restarts service once
```

This is especially powerful when tasks are independent — wall-clock time stays the same as a single task, but you get 3x the output.

## Context Budget Guidelines

| Worker Type | Target Context | Max Context |
|-------------|---------------|-------------|
| analysis-agent | < 500 tokens | 2k |
| research-agent | < 1k tokens | 5k |
| dev-agent | < 2k tokens | 8k |
| complex dev-agent | < 5k tokens | 15k |

If your task prompt exceeds these budgets, it's a sign the task isn't scoped tightly enough.

## Anti-Patterns

**❌ Giving workers full system context**  
Don't dump memory files, conversation history, or unrelated code into the task prompt. Workers only need what's relevant to their specific task.

**❌ Workers making production changes autonomously**  
Workers should report what they'd do or do the surgical change. The orchestrator applies service restarts, notifies users, and makes the call on risky changes.

**❌ Chaining workers without orchestrator review**  
Worker A's output should flow through the orchestrator before becoming Worker B's input. This maintains oversight and catches errors.

**❌ Long-running stateful workers**  
If a worker needs to remember state across multiple operations, it's doing too much. Break it into smaller tasks.
