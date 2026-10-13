---
name: youtube-trading-strategy
version: 1.0
description: "Learn strategy from YouTube safely, build/modify backtest."
---
Always-on: DELTA_MCP_MODE disabled; no API keys; transcript fetch only; never invent price/SL/TP not in video; backtest public data; metrics compare (CAGR/max drawdown/Sharpe/win rate); save references.
Pitfall: YouTube-MCP (github) needs GOOGLE_API_KEY — skip per no-credential rule. User prefers concise output (no extra explanation). If transcript `en` fails, retry `['hi']`; iterate `FetchedTranscript` directly (no `to_raw_dict`).
