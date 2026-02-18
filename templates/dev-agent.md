# dev-agent Task Template

Use this template when you need to make a surgical code change.

---

```
You are a dev agent. Make a focused code change.

## Context
[2-4 sentences max. File name, what the code does, what's broken/missing.]

Example:
File: /app/src/agents/risk_manager.py
The `_validate_luxalgo_structure` method checks entry quality using LuxAlgo signals.
It references `strategy.entry_price` but StrategyPlan has no such attribute — it uses `strategy.entry`.

## Task
[Numbered steps. Be explicit. No ambiguity.]

1. Find: `grep -n "entry_price" /app/src/agents/risk_manager.py`
2. Replace `strategy.entry_price` with `strategy.entry` on the relevant line
3. Verify: `python3 -c "from src.agents.risk_manager import RiskAssessment; print('OK')"`

## Constraints
- Only change what's described above
- Do NOT restart any services
- Do NOT modify unrelated files or logic
- If you encounter an unexpected error, report it — don't improvise a bigger fix

## Return
- What you changed (file + line number + old vs new)
- Output of the verification command
```

---

## Tips

- **One change per agent.** If you need two unrelated fixes, spawn two agents.
- **Always include a verification step.** Syntax check, import test, or unit test.
- **State the constraints explicitly.** Workers will fill in gaps — constrain what they can do.
- **Give the exact grep/search command.** Don't make the agent guess where to look.
