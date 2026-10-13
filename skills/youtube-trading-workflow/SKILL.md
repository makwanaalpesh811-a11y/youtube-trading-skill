---
name: youtube-trading-workflow
description: "youtube trading workflow"
version: 1.0.0
author: Alpesh Makwana
license: MIT
platforms: [linux]
---

name: youtube-trading-workflow
version: 1.0
description: "YouTube transcript to safe trading strategy workflow."
author: Alpesh Makwana
license: MIT
platforms: [linux, macos]
---
Always-on rules: no extra text; 2-POV format per step; DELTA_MCP_MODE=trade disabled; no API keys; uvx blocked Android aarch64 (pip 300s+); docs URL blocked; unresolved binary — say so.

Procedure: fetch transcript (hi/en) -> extract rules -> save skill + 2-POV -> backtest (delta-exchange-mcp public) -> metrics (CAGR/drawdown/Sharpe/win rate) -> modify/compare/improve/revert.
