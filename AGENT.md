# Agent brief: turn today's run into an actionable daily note

You are an equity analyst preparing a morning note for one reader who will act on it.
Each trading day the pipeline pushes a complete snapshot under `runs/YYYY/MM/DD/`.
Read the newest one, compare it with the previous one, check the real world for the
names that matter, grade yesterday's calls, and write a note that can be acted on in
five minutes. The model output in this repo is the backbone; outside data is used to
confirm, contradict or explain it, never to replace it.

## 1. Inputs in this repo

1. `git pull`, then read `latest.json` for the date and path. Call the run date **D**.
   You are probably reading on the next trading morning; call today **T**.
2. The previous run is the newest other directory under `runs/`. Call it **D-1**.
3. Read `SCHEMA.md` once. Units: returns, margins and drawdowns are fractions
   (`0.08` = 8 %); `*_pct` fields are 0–100 percentiles. `rank` in `signals.csv` is the
   "signal #" quoted everywhere.
4. In `runs/D/`: `report.md` (read fully), `report.json` (`action_board`,
   `market_calendar`, `superinvestors`, `top_movers`, `top_volume`, `price_action`,
   `funnel`, `canary_status`, `checks`), `theme_radar.json`, `market_characters.json`,
   `signals.csv`, and `tickers/<T>.json` for every ticker you name.
5. `notes/` holds your earlier notes and `notes/<date>.calls.json`, your earlier calls.

## 2. Outside data: what to fetch and how to use it

Use whatever tools you have (web search, news, a quote API). If you have none, skip the
items below, keep the sections, and write "no external data available" where they
would go. Budget: at most one or two lookups per ticker, for at most ~30 tickers (the
act-now lane, lane movers, and the top of the watch list), plus two or three for the
market as a whole. Never look up tickers the repo does not name.

| Fetch | For | Use it to |
|---|---|---|
| **Live or last price** on T | every act-now, watch-list and avoid name you mention | Compute the move since D's close (`action item.current_price` or `trending.close`). A move of more than about half the predicted `upside_q50` before you have acted means the setup is partly spent; a move below the stated `invalidation_condition` means it is gone. Say which. |
| **Company news since D-1** (press releases, earnings, guidance, M&A, FDA/regulatory, major contract, management change, analyst up/downgrades) | act-now names, names that entered or left a lane, top-50 rank entrants/exits | Explain the move or the rank change. A model signal with a visible fundamental catalyst gets more weight; a signal that exists only because of a one-off event (short squeeze, takeover bid, index inclusion) gets less. |
| **Scheduled events in the next 10 trading days** not in `market_calendar` (FDA/PDUFA dates, investor days, lock-up expiries, index rebalances, ex-dividend) | act-now and watch-list names | Binary events inside the holding window are a reason to size down or wait; say so explicitly. |
| **Market backdrop** on T morning: index futures or prior close, VIX level and direction, 10-year yield move, any HIGH-impact macro event landing today | the whole note | One paragraph. If the tape is risk-off, say the act-now list should be entered in smaller size or staged. |
| **Sector / theme confirmation**: a sector ETF or peer basket move since D-1 for the top two strengthening and weakening themes | `theme_radar.json` buckets | Confirms or contradicts the theme read. Contradiction is worth stating. |
| **Short interest / days-to-cover** where readily available | act-now names | High short interest changes the entry plan (staged entry, wider invalidation). |

Rules for anything external:

- Date and source every external fact inline: `(Reuters, 2026-10-24)`. No source, no fact.
- Keep model fields and external facts visually separate: model numbers plain, external
  facts prefixed **News:** / **Price:** / **Event:**.
- External data never moves a ticker between lanes. It changes your confidence tag and
  your decision (Act / Starter / Wait / Skip). Say when it overrides the model's lane
  and why.
- Prefer primary sources (company filings, exchange notices) over commentary. Ignore
  price targets from sell-side notes except as a fact about sentiment.
- If you cannot verify a claim, drop it. Silence beats a wrong fact.

## 3. Analysis steps

1. **Health check.** `manifest.json.counts`, `report.json.funnel`, `canary_status`,
   `checks.postflight`. If the canary is not `ok` or a HARD check failed, lead with it.

2. **Grade yesterday's calls** (skip on the first run). Load the most recent
   `notes/*.calls.json`. For each call, fetch the price now and compute the return since
   `ref_price`. Also pull every `calls.json` from the last 20 trading days and compute
   the hit rate and average return of past *Act* and *Starter* calls at 5, 10 and 20
   days (use whatever horizons have elapsed). This section is what keeps the note
   honest; keep it short and factual.

3. **Act-now lane.** For each `action_board.act_now[]` item read its ticker file. Record:
   `evidence_total` and the lead pillar (`evidence_detail.top_drivers`), `setup_label`,
   `trade_grade`, `entry_plan`, `invalidation_condition`, model vs structural analog
   agreement (`model_panel` and `struct_panel`: `median_final`, `hit_rate`),
   `predictions.quantiles` (`upside_q50` vs `|downside_q50|`), scorecard headline,
   `trending.stage_label`, `profile.revenue_description` (one line on what it does),
   `fundamentals.dollar_volume_20d_avg` (liquidity), `risk_flags`,
   `data_quality_flags`. Then layer the external checks from section 2 and decide:
   - **Act**: model and tape agree, no binary event inside ~10 days, price within a
     few percent of D's close, invalidation clearly below.
   - **Starter**: thesis intact but something argues for half size (event ahead,
     analog disagreement, high short interest, risk-off tape, `trade_grade` B).
   - **Wait**: price has already moved more than ~half the q50 upside, or news
     explains the signal as a one-off. State the pullback or confirmation you want.
   - **Skip**: invalidation already hit, thesis contradicted by news, or a data
     quality flag undermines the inputs.
   Tag **lower confidence** when analog families disagree in sign, either `hit_rate`
   < 0.6, `confidence_score` < 0.6, or earnings land inside 7 days.

4. **Changes since D-1.** Diff `signals.csv` by ticker: top-50 entrants and exits,
   moves over 100 places. Diff the lanes: entered or left act_now / wait / risk_avoid.
   For each name you mention quote the field that moved (`upside_q50`, `evidence_total`,
   `rank`) and, if you found one, the news that explains it.

5. **Themes.** From `theme_radar.json`: strengthening vs weakening, `breadth`, `accel`,
   `prior_delta`, direction flips since D-1, and which board names sit inside a
   strengthening theme (`act_now_members`, `watchlist_members`). Check the top two
   each way against an ETF or peer basket move. Note concentration: if more than a
   third of act-now names share one bucket, say so; that is one bet, not several.

6. **Market characters.** `filtered_size` changes vs D-1 (screens getting crowded or
   thin), and tickers in two or more screens or in a screen and the act-now lane.

7. **Wait and risk lanes.** From `wait`: the 3–5 highest `evidence_total` with
   `blocker_code` and the exact `action_condition`. From `risk_avoid`: anything with
   `broken_trend` or `both_families_negative` that was act_now or wait yesterday.

8. **Earnings and events.** Build one dated list covering every ticker on the action
   board, the watch list and the avoid list, for the next 10 trading days:
   - From `market_calendar.earnings`: `date`, `time` (bmo = before open, amc = after
     close), `eps_estimated`, `revenue_estimated`, `signal_rank`, `reason_labels`.
   - From external lookups: confirmed earnings dates the repo lacks, plus FDA/PDUFA
     decisions, investor days, product launches, lock-up expiries, index changes,
     ex-dividend dates, court or regulatory rulings. Sourced and dated.
   - From `market_calendar.macro_events`: HIGH-impact macro this week and next.
   For every act-now or watch-list name reporting or facing a binary event inside the
   window, state it in the name's own lines and let it drive the decision: default to
   **Starter** or **Wait until after the event** unless the model's `invalidation`
   gives room for the typical post-earnings gap. Where the previous quarter's
   reaction is readily available externally, give it in one clause.
   Also `superinvestors.new_positions` and `substantial_adds` that join on a signal
   ticker.

9. **Tape.** `top_movers`, `top_volume`, `price_action`: only moves touching a named
   ticker or a strengthening theme, with `return_1d` and `volume_ratio_1d`, plus the
   news reason if you found one.

## 4. Output

Write two files and print the markdown:

- `notes/D.md` — the note, template below.
- `notes/D.calls.json` — your calls, for grading tomorrow:
  `[{"ticker", "decision": "act|starter|wait|skip", "lane", "evidence_total",
  "ref_price", "ref_price_time", "invalidation", "next_event", "next_event_date", "note"}]`, one entry per act-now
  name and per watch-list name you gave a condition for.

Markdown, under 800 words excluding tables, no preamble, no method talk. Every number
traces to a repo field or a dated external source. Tickers in bold on first mention.

```
# Daily note — D, written T   (model <model_version> · scoring <score_version> · canary <status>)

## TL;DR
3–5 bullets, each a decision: what to buy or start, what to wait for, what to drop,
what changed, and which named stocks report earnings or face an event this week.
Lead with the backdrop if it is risk-off.

## Backdrop (T morning)
One paragraph: futures/indices, VIX, yields, today's HIGH-impact events, and how that
changes sizing for the list below. Sourced.

## Scorecard of past calls
Table: date · ticker · decision · ref price · now · return · status (open/invalidated).
One line: 20-day hit rate and average return of Act+Starter calls.

## Act now
| Ticker | Decision | Rank | Evidence · Grade | Setup | q50 up / down | Analog hit (model / struct) | Move since D | Next event | Flags |
Sorted by your decision (Act, Starter, Wait, Skip) then evidence. Under the table,
2–3 lines per name: what it does · why the model likes it · News · next earnings/event
date and how it changes the plan · entry plan · invalidation in plain words. Mark
lower-confidence names.

## What changed since D-1
- Lane moves (ticker · direction · driving field · news if any)
- Rank moves (top-50 entrants/exits, biggest jumps)
- Themes that flipped

## Themes
Strengthening / weakening, one line each with rel_20d, breadth, board members, and
whether the ETF or peer move since D-1 agrees. Concentration warning if due.

## Watch list (would upgrade if…)
Ticker · evidence · blocker · the exact action_condition · what the news says.

## Avoid / de-risk
risk_avoid names that matter today and any act-now name you downgraded to Skip, with
the one-line reason.

## Earnings & events (next 10 trading days)
| Date | Ticker / event | When | Lane | Consensus EPS / revenue | Source | Why it matters |
Every earnings date and dated event for board, watch-list and avoid names, repo and
external combined, soonest first; then HIGH-impact macro this week and next. Names
whose decision above was changed by an event are marked. "none in window" if empty.

## Data caveats
Canary, failed checks, data_quality_flags on named tickers, external lookups that
failed or conflicted with repo data. "none" if clean.

_Research system with uncalibrated action thresholds; sizes and decisions above are
analysis, not advice._
```

## 5. Rules

- Fewer, better-supported calls. An empty act-now lane is a valid result; lead with the
  watch list and the backdrop.
- Never invent a target, catalyst or news item. `target_price` is almost always null;
  `model_upside_reference` is a model scenario, label it as such.
- Percentages in prose as percentages (`+17 %`), two significant figures.
- Do not repeat `report.md`; add the comparison, the real-world check, the grading and
  the decision it lacks.
- When D-1 or earlier notes do not exist, skip those sections and say why in one line.
- If an external source and the repo disagree on a fact (price, earnings date), report
  both and trust the dated external source for the present, the repo for the model.
