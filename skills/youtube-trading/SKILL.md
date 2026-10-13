---
name: youtube-trading
description: "Learn trading from video transcripts; build safe backtests."
---
Always-on: DELTA_MCP_MODE=disabled; DELTA_API_KEY never; no live orders. Minimal output, no filler. User prefers Hindi/English mixed, 2-point-of-view review (self + work), step-by-step, bounded execution (no infinite loops), 3-part response format (1-Kya mila, 2-Update, 3-Status). Procedure: transcript fetch -> extract rules (entry/exit/SL/TP/duration/delta) -> save -> backtest (delta-exchange-mcp public: get_candles/get_ticker/get_options_chain) -> metrics (CAGR/max-drawdown/Sharpe/win-rate) -> compare -> modify/revert.

HARD RULE: Before claiming 'no data / proxy needed', INSPECT ALL available files first. Delta data has BOTH futures (ETHUSD_YYYY-MM.csv) AND options (ETH_YYYY-MM.csv) ticks. Options file format: `{C|P}-ETH-{STRIKE}-{DDMMYY}` columns product_symbol,price,size,timestamp,buyer_role. ATM strike = min |call_price - put_price| at entry time (put-call parity). Greeks NOT required for SL/trail/target backtests on raw option prices. Only claim proxy when data is genuinely absent after listing both CSVs.

## Grid winner selection (never pick by raw total PnL)

When a grid tests many configs, rank by a PASS gate that separates genuine edge from outlier-driven configs:

- **ex-top-3 PnL**: sum of all trades' PnL minus the 3 largest winning trades. If this is negative or only barely positive, the config's headline total is carried by 3 lucky trades — reject it even if total PnL is highest.
- **combined MaxDD**: equity curve including all trades (sell-leg + lazy-leg combined if applicable), not just lazy-leg alone.
- **PASS gate**: `total >= BASE.total AND combined_MaxDD <= BASE.MaxDD AND ex-top3 >= BASE.ex-top3 AND worse_trade_frac <= 0.35 AND months_better_frac >= 0.6`. Report configs that pass ALL of these; if none pass, show the top by ex-top-3 and say so explicitly.
- **Report both halves**: split months into first half / second half; a config that wins one half and loses the other is regime-dependent — flag it, don't rank it as a winner.

## Multi-month carry-forward for ATM lookback

When entry requires an ATM strike (put-call parity) computed from a prior tick, the current month's CSV alone is insufficient at the month boundary. Load each month with its prior month as `--prev`, restricted to the last 9 hours before the first tick of the current month. This makes day-01 entries use real ATM strikes instead of the fallback.

## Multi-leg trigger pattern (lazy-leg)

For strategies that add a leg only after the primary leg stops out:
- primary leg = ATM short (both CE and PE for iron condor / straddle)
- trigger = primary leg premium >= entry * (1 + SL_pct) — e.g. 2.3 = 130% SL
- lazy leg = buy ATM on the SAME side (call/put) as the stopped-out primary leg, at the time of the stop
- recompute ATM strike at trigger time (underlying has moved; original ATM is stale)
- test trail/SL/target only on the lazy-leg segment between trigger and session exit

References: references/strict-backtest-rules.md, references/video-analysis.md, references/lazy-leg-grid-backtest.md.
