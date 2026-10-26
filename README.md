# InflectSignals daily runs

Machine-written snapshot of each trading day's InflectSignals run: the Daily Action
Report, the full ranked signal universe, and a detail file for every ticker the
report mentions. Pushed automatically after the nightly pipeline finishes.

Everything is plain text (Markdown, CSV, JSON). `SCHEMA.md` documents every file and
field once; the data files carry no inline documentation. `AGENT.md` is the brief for an
AI agent that turns a day's run into an actionable note (written to `notes/YYYY-MM-DD.md`).

## Layout

```
latest.json                      → {"date": "YYYY-MM-DD", "path": "runs/YYYY/MM/DD"}
notes/YYYY-MM-DD.md              agent-written daily note (see AGENT.md), if any
runs/YYYY/MM/DD/
  manifest.json                  run metadata, counts, sha256 of every file
  report.md                      Daily Action Report, human-readable
  report.json                    Daily Action Report, complete structured data
  theme_radar.json               industry-theme momentum table (extracted from report.json)
  market_characters.json         market-character screens + their definitions (extracted)
  signals.csv                    every primary signal, ranked 1..N
  tickers/<TICKER>.json          detail page for each report-referenced ticker
```

## How to read a day

1. Open `latest.json` for the most recent date, or pick a `runs/YYYY/MM/DD/`.
2. `report.md` is the narrative: market calendar, theme radar, market characters,
   superinvestors, then the action board (**Act now** / **Wait for price-proof** /
   **Risk / avoid**).
3. `report.json` has the same content with every number, including the per-ticker
   scoring breakdown (`action_board.*[].evidence_detail`).
4. `signals.csv` ranks the whole universe (about 2,100 tickers). `rank` is the
   "signal #" quoted in the report. `report_lane` tells you if a row made the action
   board, `has_detail_file` whether `tickers/<T>.json` exists for it.
5. `tickers/<T>.json` holds what the dashboard's signal detail page shows: company
   profile, fundamentals, quality scorecard, model predictions, detected regions,
   momentum ("Trending"), model analogs, structural analogs, feature attribution,
   peer group, market characters.

## Conventions

- Returns, margins, growth rates, yields and drawdowns are **fractions**
  (`0.0812` = 8.12 %), never percent, unless the key ends in `_pct` or `_pctile`
  which are 0–100 percentiles, or the value is a labelled string.
- Prices are USD. `market_cap_m` is USD millions. Dates are `YYYY-MM-DD`.
- Floats are rounded to 4 decimals. Missing values are `null` (JSON) or empty (CSV).
- JSON keys are sorted; files are stable day to day so `git diff` is meaningful.
- Model outputs are forward-looking estimates from an uncalibrated research system.
  Nothing here is investment advice.

## Compare days

`git log --stat -- runs/` lists runs. Diff two days' rankings:

```bash
diff <(cut -d, -f1,2 runs/2026/10/22/signals.csv) <(cut -d, -f1,2 runs/2026/10/23/signals.csv)
```

Track a ticker: `git log -p -- 'runs/*/*/*/tickers/FTNT.json'`.
