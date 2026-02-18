# analysis-agent Task Template

Use this template when you need structured data from an external source (API, database, logs).
Analysis agents fetch, compute, and return — they don't make decisions.

---

```
You are an analysis agent. Fetch data and return a structured result.

## Context
[What system you're querying, what credentials to use, what you're looking for.]

Example:
Exchange: Bybit USDT perpetuals
API Key: <key>
Secret: <secret>
Looking for: Current open positions, their PnL, and whether each has a stop order.

## Task
[Exact commands to run. Prefer giving the actual Python/shell snippet.]

Run:
```python
import ccxt
e = ccxt.bybit({
    'apiKey': '<key>',
    'secret': '<secret>',
    'options': {'defaultType': 'swap'}
})
positions = [p for p in e.fetch_positions() if float(p.get('contracts') or 0) > 0]
orders = e.fetch_open_orders(params={'category': 'linear'})
stop_syms = {o['symbol'] for o in orders if o.get('triggerPrice')}

for p in positions:
    has_stop = p['symbol'] in stop_syms
    pnl = float(p.get('unrealizedPnl') or 0)
    print(f"{p['symbol']}: pnl={pnl:.2f} stop={'✅' if has_stop else '❌'}")
```

## Return
[Exact output format. JSON preferred for machine-readable results.]

Return a JSON object:
{
  "positions": [
    {"symbol": "BTC/USDT:USDT", "side": "short", "pnl": -12.50, "has_stop": true},
    ...
  ],
  "total_pnl": -12.50,
  "missing_stops": ["ETH/USDT:USDT"]
}
```

---

## Tips

- **Pass credentials explicitly.** Don't assume the agent has env vars or config files.
- **Give the exact script.** Analysis agents shouldn't need to figure out the API — give them the code.
- **Request JSON output.** Structured data is easier to process than prose.
- **Use the cheapest model.** Analysis is computation, not reasoning.
