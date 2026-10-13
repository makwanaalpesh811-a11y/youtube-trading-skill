Backtest rules (user-enforced):
- Strict separation: video rules vs analyst assumptions in separate tables.
- Missing video-spec rules = "Video mein specified nahi hai"; never invent defaults.
- Proxy backtests allowed ONLY with clearly labeled assumption table; never present as true historical.
- Historical option-chain unavailable via Delta MCP public -> block backtest, do not invent.
- LOCAL DATA FIRST: before claiming option data is missing, `ls ~/ | grep -iE 'ETH|BTC'` + `ls ~/delta_candles/` — futures CSVs (`ETHUSD_YYYY-MM.csv`), options CSVs (`C-ETH-*.csv` / `P-ETH-*.csv`), and JSON candles (`delta_candles/*_1h.json`) may already exist. Prefixes like `res_`/`nat_`/`v5_` ETH files are often daily-PnL outputs of prior backtests, NOT raw option-chain data — check content before assuming.
- When user asks for profitable settings and data shows all-negative PnL, report best (least-loss) variant explicitly; do not fabricate profitable parameters. Offer labeled proxy assumption table only if explicitly requested.
- No live trading; DELTA_MCP_MODE never enabled; no credentials exposed.
