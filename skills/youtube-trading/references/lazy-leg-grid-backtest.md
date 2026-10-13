# Lazy-leg grid backtest — reference recipes

Durable reference for the multi-month options backtest on Delta's historical tick data. Not the workflow (that's in the parent SKILL.md) — this file holds the specific recipes that are safe to copy.

## Data layout

- Path: `/sdcard/Hermes_Files/delta_data/extracted/`
- Files: `ETH_YYYY-MM.csv` (options) and `ETHUSD_YYYY-MM.csv` (futures), one per month
- Options columns: `product_symbol, price, size, timestamp, buyer_role`
- Symbol format: `{C|P}-ETH-{STRIKE}-{DDMMYY}` (e.g. `C-ETH-1900-010826` = call, strike 1900, expiry 01 Aug 2026)
- Timestamps are UTC; entry/exit times (22:00/07:00 IST) = 16:30/01:30 UTC

## ATM strike via put-call parity

```python
def atm(expiry, t, maxage=1800):
    best = None
    for K in sorted(STR[expiry]):
        c = last_before(f'C-ETH-{K}-{expiry}', t, maxage)
        p = last_before(f'P-ETH-{K}-{expiry}', t, maxage)
        if c is None or p is None: continue
        if best is None or abs(c - p) < best[0]:
            best = (abs(c - p), K)
    return best[1] if best else None
```

`maxage=1800` (30 min) — stale option quotes are worse than no quote. Returns `None` if no expiry has both call and put within the window.

## Three trail rule families

```python
# Point-based: SL is N points below entry; trail moves it up
# f(entry, peak) -> stop price
def pts(slp, x=None, y=None):
    def f(e, m):
        s = e - slp
        if x:
            s = e - slp + y * int((m - e) / x + 1e-9)
        return s
    return f

# Percent-based with lock + ratchet: SL slp% below entry; when peak reaches
# entry*(1+trig), lock in `lock` and boost SL by `inc` per `step` of gain
def pct(slp, trig=9e9, lock=0., step=None, inc=None):
    def f(e, m):
        s = e * (1 - slp); g = (m - e) / e
        if g >= trig:
            l = lock + (inc * int((g - trig) / step + 1e-9) if step else 0)
            s = max(s, e * (1 + l))
        return s
    return f

# Percent trail (AlgoTest-native style): SL slp% below entry; every x% gain
# adds y% to SL. Constraint y <= x, otherwise trail is a loss.
def ptrail(slp, x, y):
    def f(e, m):
        return e * (1 - slp) + e * y * int(((m - e) / e) / x + 1e-9)
    return f
```

## Grid template

```python
def build_rules():
    R = {BASE: pts(15, 1, 1), 'HOLD no SL': lambda e, m: -1e9}
    for sl in (10, 15, 20, 30, 40, 60):
        R[f'SL{sl}pt only'] = pts(sl)
        for x, y in ((3,3),(5,5),(10,5),(10,10)):
            R[f'SL{sl}pt trail{x}/{y}'] = pts(sl, x, y)
    for sl in (.2,.3,.4,.5,.6,.7,.8,1.0):
        R[f'SL{int(sl*100)}% only'] = pct(sl)
        for trig in (.2,.3,.4,.5,.75,1.0):
            for lock in (0,.05,.1,.2):
                R[f'SL{int(sl*100)}% lock{int(lock*100)}@{int(trig*100)}%'] = pct(sl, trig, lock)
            for step in (.25,.5,1.0):
                for inc in (.05,.1,.25):
                    R[f'SL{int(sl*100)}% lock10@{int(trig*100)}% +{int(inc*100)}%/{int(step*100)}%'] = pct(sl, trig, .1, step, inc)
        for x in (.1,.2,.25,.3,.5):
            for y in (.05,.1,.2,.25,.3,.5):
                if y <= x + 1e-9:
                    R[f'SL{int(sl*100)}% trail{int(x*100)}%/{int(y*100)}%'] = ptrail(sl, x, y)
    return R
```

## Report skeleton (rank by PASS gate)

```python
def report(sell_by_day, trades):
    days = sorted(sell_by_day)
    months = sorted(set(d[:7] for d in days))
    def stats(c):
        by = {}
        for t in trades:
            by[t['day']] = by.get(t['day'], 0) + t['pnl'][c]
        eq = pk = dd = 0
        for d in days:
            eq += sell_by_day[d] + by.get(d, 0)
            pk = max(pk, eq); dd = max(dd, pk - eq)
        p = sorted((t['pnl'][c] for t in trades), reverse=True)
        return sum(p), dd, sum(p) - sum(p[:3])

    bt = [t['pnl'][BASE] for t in trades]
    bm = {m: sum(t['pnl'][BASE] for t in trades if t['day'][:7] == m) for m in months}
    b_tot, b_dd, b_x3 = stats(BASE)
    rows = []
    for c in trades[0]['pnl']:
        tot, dd, x3 = stats(c)
        n = len(trades)
        imp = sum(1 for t,b in zip(trades, bt) if t['pnl'][c] > b + .05) / n
        wor = sum(1 for t,b in zip(trades, bt) if t['pnl'][c] < b - .05) / n
        mb  = sum(1 for m in months if sum(t['pnl'][c] for t in trades if t['day'][:7] == m) >= bm[m] - 1e-9)
        ok  = tot >= b_tot and dd <= b_dd + 1e-9 and x3 >= b_x3 and wor <= .35 and mb >= .6 * len(months)
        rows.append((ok, x3, tot, dd, imp, wor, mb, c))
    ok_rows = sorted([r for r in rows if r[0]], key=lambda r: (-r[1], -r[2]))
    # print top ok_rows first, then BASE, then best non-passing by ex-top-3
```

## Rupee conversion

Delta ETH options tick size = 0.01 ETH; USD = tick unit. For a 0.01 ETH position, 1 point ≈ 0.85 rupees. Multiply config `total` × 0.85 for rupee/lot.

## Session time zones

Delta data timestamps are UTC. Algotest UI displays IST (UTC+5:30). Entry 22:00 IST = 16:30 UTC same day; exit 07:00 IST = 01:30 UTC next day (9-hour session).

## Session boundary handling

When a session starts near the beginning of a month (e.g. Aug 01 16:30 UTC), the ATM strike needs prior ticks from July 31. Load the previous month's CSV with a `tail_from` filter to include only the last 9 hours before the current month starts:

```python
def do_month(current_path, prev_path):
    load(current_path)
    if prev_path:
        load(prev_path, tail_from=MINT[0] - 9 * 3600)
```
