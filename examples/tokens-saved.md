# Real Token Savings — Live Examples

All examples are from a real session building a multi-agent AI system.

The main session had accumulated ~50k tokens of context at this point (conversation history, memory files, code reads, tool results).

---

## Example 1: Codebase Structure Analysis

**Task:** Map the project directory structure — list subdirectories, count agent files, find entry point.

| Approach | Tokens In | Tokens Out | Total | Time |
|----------|-----------|-----------|-------|------|
| Sub-agent (fresh context) | 4 | 340 | **344** | 6s |
| Main session (50k context) | ~14,000 | ~500 | ~14,500 | — |
| **Savings** | | | **97.6%** | |

Result was identical. The agent ran 3 shell commands and returned a JSON object.

---

## Example 2: Bug Fix — Wrong Attribute Name

**Task:** Fix `AttributeError: 'StrategyPlan' object has no attribute 'entry_price'` — change to `strategy.entry`.

| Approach | Tokens In | Tokens Out | Total | Time |
|----------|-----------|-----------|-------|------|
| Sub-agent | 7 | 1,100 | **1,107** | 22s |
| Main session | ~13,000 | ~800 | ~13,800 | — |
| **Savings** | | | **92.0%** | |

Agent found the line, made the edit, ran syntax check, reported back. One file, one change.

---

## Example 3: Root Cause Investigation — Stale Price Ranges

**Task:** Why is DOT/USDT failing price validation? Find the hardcoded ranges, get the current actual price, recommend a fix.

| Approach | Tokens In | Tokens Out | Total | Time |
|----------|-----------|-----------|-------|------|
| Sub-agent | 8,700 | 935 | **9,635** | 49s |
| Main session | ~38,000 | ~1,500 | ~39,500 | — |
| **Savings** | | | **75.6%** | |

Note: This agent had a larger context because the task required reading multiple files. Still 3.7x cheaper.

---

## Example 4: Log Rotation Setup

**Task:** Install logrotate config for a 12GB log file — create config, dry-run test, verify.

| Approach | Tokens In | Tokens Out | Total | Time |
|----------|-----------|-----------|-------|------|
| Sub-agent | 4 | 581 | **585** | 13s |
| Main session | ~14,000 | ~800 | ~14,800 | — |
| **Savings** | | | **96.0%** | |

---

## Cumulative Savings (This Session)

| Metric | Value |
|--------|-------|
| Sub-agents spawned | 4 |
| Total sub-agent tokens | ~11,671 |
| Estimated main session equivalent | ~82,600 |
| **Total savings** | **~85.9%** |
| **Approximate cost savings** | **~$0.50–$2.00** (depending on model pricing) |

---

## Parallelism Bonus

Two of the above tasks ran simultaneously (price validation research + codebase analysis). Wall-clock time: same as one task. Output: double.

With a traditional single-agent approach, these would have run sequentially, and each would have carried the full accumulated context.

---

## Key Takeaway

The savings compound as the main session grows. A fresh sub-agent always starts at ~0 tokens of context. The main session never does.

The longer a session runs, the more valuable the pattern becomes.
