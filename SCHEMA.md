# Schema (version 1)

Every file under `runs/YYYY/MM/DD/` and every field group, documented once.
Units: returns, margins, growth, yields and drawdowns are fractions (`0.08` = 8 %)
unless the key ends in `_pct`/`_pctile`/`universe_pct`/`sector_pct` (0–100) or the
value is a text label. Prices USD, `market_cap_m` USD millions, dates `YYYY-MM-DD`.

## Vocabulary used throughout

| Term | Meaning |
|---|---|
| **region** | A detected price pattern on one ticker: starts at `region_start`, confirmed on `trigger_date`. The model scores the ticker from the pattern's state on the run date. |
| **trigger_type** | Pattern family, fixed at detection: `runup`, `drawdown`, `shallow_drawdown`, `severe_drawdown`, `compression`. |
| **primary signal** | A ticker's most recent region; the one that is ranked. Older open regions are *secondary*. |
| **signal rank** | 1..N over all primary signals, by `rank_score` descending (tie-break confidence). This is the "signal #" quoted in the report. N = `funnel.signal_universe_size`. |
| **rank_score** | confidence × calibrated expected value × log(1 + asymmetric risk/reward). Composite signal quality. |
| **ev_score** | Expected value of the forward return implied by the predicted quantiles. |
| **confidence_score** | Model certainty 0–1. |
| **upside_qXX / downside_qXX** | Predicted quantiles (10/25/50/75/90) of the maximum favourable / adverse excursion over the model horizon. Upside quantiles are calibrated; `*_raw` where present is the uncalibrated model output. |
| **model analogs** | Historical regions whose learned embedding is nearest to this one (`historical_matches`), with what actually happened afterwards. |
| **structural analogs** | Historical regions matched on price-path shape and context (`structural_matches`), independent of the model's embedding. |
| **panel** | Robust summary of an analog set: `n`, `n_eff`, `median_final`, `p25_final` (final-return median/25th pct), `hit_rate` with `hit_lo/hit_hi` CI, `median_pain` (median adverse excursion), `p_pain_gt5` (share with >5 % pain), `return_to_pain`, `edge`. |
| **gp_core_ratio** | log(1+avg upside)/log(1+avg downside) over the core structural analogs; `gp_core_pct` is its percentile across today's universe. >1 means gains outweighed pain historically. |
| **market characters** | Deterministic screens `g1…g13` (Steady Climbers, …) defined in `market_characters.json.registry`. |
| **lane** | Action board placement: `act_now`, `wait` (wait for price-proof), `risk_avoid`. |

## `manifest.json`

| Key | Meaning |
|---|---|
| `schema_version` | This document's version. |
| `date` | Run date (last completed US trading day the run describes). |
| `generated_at` | UTC timestamp of the export. |
| `report_version`, `score_version`, `model_version` | Stage 8 report format (`r1`), action-scoring version (`action-v7`), active model (`gru_v004`). |
| `funnel` | `primary_signal_count` → `eligible_count` → `act_now_count` / `wait_count` / `risk_avoid_count`; `signal_universe_size` = N for signal rank. |
| `counts` | Rows in `signals.csv`, number of `tickers/*.json`, report-referenced ticker count, per-lane counts. |
| `files` | `{path: {bytes, sha256}}` for every file in the run dir. |
| `pipeline_git_sha` | Commit of the pipeline code that produced the run. |
| `source_fingerprint` | Stage 8's hash of its inputs + config; same fingerprint ⇒ identical report. |
| `stage7_api_url` | Where the detail payloads were read from. |

## `report.md`

The Daily Action Report rendered for humans. Sections in order: header (model,
versions, funnel), Market Calendar (macro events + earnings this/next week), Theme
Rotation, Top Movers / Top Volume / Price Action, Market Characters, Superinvestors,
Action Board (Act now / Wait for price-proof / Risk-avoid) with per-ticker rationale,
checks. Byte-identical to the pipeline's own `report.md`.

## `report.json`

Same content, structured. Top-level keys:

| Key | Content |
|---|---|
| `report_date`, `report_version`, `score_version`, `model_version`, `ai_status` | Identity. `ai_status` is `unavailable` when no AI narrative was produced; all scoring is deterministic regardless. |
| `funnel` | See manifest. |
| `canary_status`, `canary_detail` | `ok`, or `empty` / `empty_streak` / `universe_shrink` when the report looks degenerate. |
| `watchlist_refresh` | Housekeeping result for the dashboard watchlist (`refreshed`, `total`, per-item `results`, `error`). Not analysis data. |
| `market_calendar` | `window` (this/next week), `macro_events[]` (`date, time, time_zone, event, category, impact, previous, estimate, actual, unit`), `earnings[]` (`date, time` bmo/amc, `symbol, eps_estimated, revenue_estimated, reasons, reason_labels, signal_rank`), `min_impact`, `stale`, `errors`. |
| `superinvestors` | 13F-derived: `fresh_filings[]`, `most_owned[]`, `new_positions[]`, `substantial_adds[]`, `conviction[]` (each row: `issuer, ticker, tickers, managers, median_weight, max_weight`, plus `holder_count`/`conviction_count`), `coverage_pct`, `period_end`, `disclosure_note`. Quarter-end data published with a lag. |
| `theme_radar` | See `theme_radar.json`. |
| `top_movers` | `gainers[]`, `losers[]`: `ticker, return_1d, return_5d, return_20d, close, market_cap_m, dollar_volume_20d_avg, volume_ratio_1d, signal_rank`. |
| `top_volume` | `1d`, `3d`, `7d` lists: as above plus `volume_ratio` (window volume vs 20d avg), `return_window`. |
| `price_action` | `momentum[]` (strong 5d/20d continuation; adds `volume_ratio_5d`), `bounce_back[]` (sharp rebound off a drawdown; adds `drawdown_from_high`). |
| `market_character_registry`, `market_characters` | See `market_characters.json`. |
| `action_board` | `act_now[]`, `wait[]`, `risk_avoid[]` of **action items** (below). |
| `checks` | `preflight[]` / `postflight[]` contract checks: `check, level` HARD/SOFT, `status` PASS/WARN/FAIL, `detail`. |

### Action item (`action_board.*[]`)

| Field | Meaning |
|---|---|
| `ticker`, `region_start`, `trigger_type`, `signal_rank`, `direction` (`long`) | Identity of the signal being judged. |
| `action_state` | Lane: `act_now` / `wait` / `risk_avoid`. `lane_rank` = position within the lane, `display_rank` = position across the board. |
| `evidence_total` | 0–100 readiness score = sum of the four pillars. |
| `pillar_forecast`, `pillar_analog`, `pillar_technical`, `pillar_fundamental` | Points from model forecast, analog agreement, technical setup, business momentum. |
| `setup_label` | Technical shape on the run date: `early_trend`, `smooth_continuation`, `vol_compression`, `extended`, `countertrend`, `downtrend`, `broken_trend`. Independent of `trigger_type`. |
| `blocker_code` | Why not `act_now`: `extended`, `technical_below_min`, `standout_dimensions`, `small_cap`, `healthcare_event_risk`, or null. `blocker_condition` spells it out. |
| `risk_flags` | Any of `analog_conflict` (model vs structural analogs disagree), `both_families_negative`, `broken_trend`, `extended`, `volatile_biopharma_stock`. |
| `trade_grade` | `A`/`B` act now (A above the configured evidence floor), `C` wait, `D` risk-avoid, `F` risk-avoid with broken trend or both analog families negative. `trade_grade_basis` states it is deterministic. |
| `why_now`, `action_condition`, `invalidation_condition`, `entry_plan` | Generated rationale text. |
| `review_by`, `valid_until`, `time_horizon` | Dates the item should be re-checked / expires. |
| `current_price`, `price_change_1d` | Close and 1-day return on the run date. |
| `target_price`, `target_basis`, `target_status` | Price target only when a sourced valuation exists; usually null with an explanatory `target_basis`. |
| `model_upside_reference`, `model_downside_reference`, `model_scenario_note` | Dollar translation of the model's median upside/downside; not a target or stop. |
| `watchlist_status`, `watchlist_group`, `watchlist_priority`, `suppressed_by` | Dashboard watchlist linkage; null when not on it. |
| `evidence_detail` | The numbers behind the pillars (below). |

### `evidence_detail`

| Key | Meaning |
|---|---|
| `close`, `ev_score`, `confidence_score`, `upside_q25/q50`, `downside_q25/q50`, `rank_score_pct` | Signal-level inputs; `rank_score_pct` is percentile within the eligible universe. |
| `market_characters` | Screen ids this ticker passes. |
| `signal_strengthening` | Prior vs current region on the same ticker: `region_count`, `upside_q50_delta`, `ev_delta`, `confidence_delta`. |
| `peer_group` | `buckets` (industry taxonomy buckets), `strength_pct` (ticker's rank within peers), `median_return_30d` of peers, `member_count_min`, `return_field`. |
| `feature_attribution` | Top model drivers: `feature, importance, importance_pct, direction, method, feature_value`. |
| `score_factors` | Every scoring factor: `factor_id, pillar, label, points, max_points, min_points, direction, summary, raw_value`. Sums to `evidence_total`. |
| `top_drivers`, `key_drag` | The factors contributing most / the largest negative. |
| `model_panel`, `struct_panel` | Analog panels (see vocabulary). |
| `technical` | Stage 2 technicals on the run date: returns 1d–90d, `stock_vs_spy_20d`, `stock_vs_sector_20d`, `above_200d_ma`, `ma_200_slope`, `volatility_20d/60d`, `volume_ratio_1d/5d`, `volume_trend`, `trend_smoothness_60/_pct`, `price_vs_high_52w`, `rsi_14`, `macd_histogram`, `adx_14`, `ff_industry`. |
| `fundamental` | `revenue_growth_yoy`, `revenue_growth_accel`, `eps_growth_yoy`, `eps_growth_accel`, `fcf_margin`, `operating_margin`, `operating_margin_change_yoy`, `roe`. |

## `theme_radar.json`

`{date, themes[]}`. One row per industry taxonomy bucket with enough members:

| Field | Meaning |
|---|---|
| `bucket` | Taxonomy bucket name (e.g. "Security, Identity & Compliance Software"). |
| `direction` | `strengthening` / `weakening` (sign of the move most members share). |
| `median_return_20d` | Median 20-day return of members. |
| `rel_20d`, `rel_5d` | Median return relative to the whole universe over 20d / 5d. |
| `accel` | `rel_5d − rel_20d / 4`: positive = the last week outpaces the 20-day pace. |
| `breadth` | Share of members with positive 20d return. |
| `member_count` | Members in the bucket. |
| `top_contributors[]` | `{ticker, return_1d}` biggest 1-day movers in the bucket. |
| `prior_report_date`, `prior_delta` | Previous report and change in `rel_20d` since then. |
| `act_now_members`, `watchlist_members` | Bucket members on today's act-now lane / the watchlist. |

## `market_characters.json`

`{date, registry, groups[]}`.

- `registry[gN]`: the screen's definition — `id, label, name, description, formula`
  (score formula in words), `filters[]` (hard conditions), `required_fields`,
  `result_columns`, `score_components[]` (`key, weight, direction, condition`),
  `uses_expanded_universe`, `registry_version`.
- `groups[]`: today's results — `id, label, name, description, filters, universe_size,
  filtered_size` and `rows[]` (top 10 by `score`). Row fields: `ticker, score,
  biopharma_downranked, close, market_cap_m, signal_rank`, then the screen's own
  `result_columns` (returns, `sharpe_252d`, `drawdown_from_high_252d`, `pe_ratio`,
  `ps_ratio`, `rs_rating`, `swing_*_30d`, growth metrics, …). Keys match Stage 2
  feature names and are named by what they measure.

## `signals.csv`

One row per primary signal, sorted by `rank`. Columns:

| Column | Meaning |
|---|---|
| `rank` | Signal rank 1..N (see vocabulary). |
| `ticker`, `trigger_type`, `region_start`, `trigger_date`, `days_in_region` | The primary region. |
| `rank_score`, `ev_score`, `confidence_score` | Composite / expected value / certainty. |
| `upside_q25`, `upside_q50`, `upside_q75` | Calibrated upside quantiles. |
| `downside_q25`, `downside_q50`, `downside_q75` | Downside quantiles (negative = loss). |
| `uncertainty` | `upside_q90 − upside_q10` (inter-decile spread of calibrated upside). |
| `risk_reward` | Upside q50 / |downside q50|. |
| `quality_label` | Rule-based pattern label from backtests: e.g. `HIGH CONVICTION`, `AVOID` (shallow drawdown), or empty. |
| `conviction_label` | `HIGH CONVICTION` / `MODERATE` / … from upside and risk/reward thresholds. |
| `is_diamond` | All five best-signal criteria met: EV top 20 %, confidence top 10 %, risk_reward ≥ 1.67, gp_core_pct ≥ 80, momentum stage early/mid. |
| `passes_tradability_filter` | Market cap ≥ $1 B and 20d dollar volume ≥ $5 M. |
| `market_characters` | `;`-joined screen ids. |
| `taxonomy_sector` | Revenue-driver taxonomy sector. |
| `market_cap_m`, `return_252d`, `volatility_20d` | Size, 1-year return, 20d daily volatility. |
| `momentum_stage` | `early` / `mid` / `developed` / `sustained` (shown as "Trending") / `rundown` / empty. |
| `data_quality_flags` | `;`-joined: `market_cap_below_1b`, `dollar_volume_below_5m`, `turnover_gt_20pct_mcap`, `daily_pe_move_gt_50pct_20d`, `daily_ps_move_gt_50pct_20d`, `daily_fcf_yield_near_clip`, `revenue_qoq_shock`, `financial_filing_shock_review`. |
| `report_lane` | `act_now` / `wait` / `risk_avoid` / empty. |
| `has_detail_file` | `True` when `tickers/<ticker>.json` exists. |

## `tickers/<TICKER>.json`

Written for every ticker named anywhere in the report. Top level: `ticker`, `date`,
`signal_rank`, `report` (`{lane, lane_rank, display_rank, setup_label, why_now}` or
null when not on the action board), then one key per module:

### `profile`
Company profile (FMP) merged with local data: `companyName, description, sector,
industry, country, fullTimeEmployees, ceo, ipoDate, website, exchangeShortName, price,
beta, mktCap`, TTM ratios `peRatioTTM, priceToSalesRatioTTM,
enterpriseValueOverEBITDATTM, dividendYielTTM, returnOnEquityTTM, debtToEquityTTM`,
`_local` (same metrics from the pipeline's own fundamentals), `latest_earnings_date`,
and the revenue-driver taxonomy: `revenue_description` (what the company sells),
`revenue_themes[]` (tags), `revenue_description_themes[]` (growth narratives),
`revenue_taxonomy[]` (`bucket, sector, segment, path, matched_tags`).

### `fundamentals`
Latest Stage 2 fundamental and valuation features (one value each). Groups:
valuation `daily_pe, daily_ps, daily_pb, daily_fcf_yield, daily_earnings_yield,
pe_ratio, pb_ratio, ps_ratio, ev_to_ebitda, ev_to_fcf, ev_to_sales, peg_trailing,
pe_percentile_1y, ps_percentile_1y, enterprise_value`; profitability `roe, roa, roic,
inc_roic, roce, gross_margin, operating_margin, net_margin, fcf_margin,
*_change_yoy, margin_vs_sector_median`; growth `revenue_growth_yoy` (+ `_q2..q8`
prior quarters), `revenue_growth_qoq/accel`, `eps_growth_*`, `ebitda_growth_yoy`,
`fcf_growth_yoy`, `revenue_cagr_3y/5y`, `eps_cagr_3y/5y`; per share `eps_ttm,
eps_diluted, eps_basic, revenue_per_share, book_value_per_share, fcf_per_share`;
balance sheet `debt_to_equity, debt_ebitda, interest_coverage, current_ratio,
current_liabilities, total_assets, net_income, operating_income, share_dilution_1y`;
quality `piotroski_f_score, altman_z_score, accrual, earnings_quality`; market
`market_cap_ratio, w52_high, w52_low, price_vs_high_52w, days_since_52w_high,
price_vs_sma200, avg_volume, dollar_volume_20d_avg, amihud_illiquidity, beta_252d,
correlation_spy_60d, volatility_20d/60d, vol_regime, rsi_14, adx_14, stock_vs_spy_20d,
stock_vs_sector_20d, sector_momentum`; histories `revenue_history_8q, eps_history_8q,
gross_margin_history, quarter_labels_8q, quarterly_revenue_growth_yoy(_8q),
quarterly_eps_growth_yoy(_8q)`; `is_pre_revenue`.

### `scorecard`
`headline`: `{avg_universe_pct, label, metric_count}` where label is `strong` (≥80),
`above_average` (≥60), `mixed` (≥40) or `below_average` — this is the "Above-average
fundamentals" badge. `metrics`: per metric (`roe, roic, operating_margin, gross_margin,
revenue_growth_yoy, eps_growth_yoy, pe_ratio, debt_to_equity`) → `{value, label,
direction` higher/lower is better, `universe_pct, sector_pct, sector, sector_n,
universe_n}`.

### `predictions`
Today's model output for the primary region: `rank, rank_score, ev_score,
confidence_score, quality_label, conviction_label, is_diamond, badge_count` (count
of positive quality badges), `passes_tradability_filter, tradability_filter_reasons,
data_quality_flags`, and `quantiles` (`upside_q10..q90`, `downside_q10..q90`,
`uncertainty, risk_reward, path_volatility`).

### `detected_regions`
`primary` (`region_start, trigger_type, trigger_date, days_in_region`) and
`secondary[]` — other open regions on the same ticker with their own
`ev_score, confidence_score, upside_q25/q50/q75, downside_q50, seq_len`.

### `trending`
Momentum scanner row (null if the ticker is not in the scan): `stage` and
`stage_label` ("Trending" = sustained), `momentum_score, total_score, rundown_score,
vcp_score, vcp_detected, breakout_confirmed, rank, rank_up, rank_down`, returns
`ret_20/60/120`, `slope_20/60, acceleration, trend_ratio, range_20/50, base_high_120,
avg_vol_20d, close, momentum_start, score_date`.

### `model_analogs`
`stats` (`n_matches, win_rate, sim_weighted_win_rate, avg_similarity, top_similarity,
median_return, return_p5/p25/p75/p95, best_return, worst_return, avg_upside,
avg_trough, avg_max_drawdown, top5_avg_upside, top5_avg_trough, upside_range,
downside_range, years_span, outcome_agreement`), `matches[]` (top 10 by similarity:
`ticker, trigger_type, start_date, trigger_date, end_date, end_reason, days_in_region,
similarity, actual_return, actual_upside, actual_downside`), `total_matches` (50 in
the full set).

### `structural_analogs`
`stats` (`n_matches, avg_similarity, top_similarity, win_rate, median_return,
p25_return, avg_upside, top5_avg_upside, avg_trough, median_pain, p_pain_gt5,
return_to_pain, top5_avg_trough`), `panel` (see vocabulary), `gp_core_ratio`,
`gp_core_pct`, `matches[]` (top 10: model-analog fields plus `total_distance`,
`seq_len`), `total_matches` (about 200 in the full set).

### `feature_attribution`
`top_drivers[]` (`feature, importance, importance_pct, direction, method,
feature_value`), `macro_driver_count`, `risk_flags`.

### `peer_group`
`buckets[]` and `groups{bucket: rows[]}`: up to 10 closest peers per taxonomy bucket
by revenue-driver similarity, each `ticker, revenue_driver, return_7d, return_30d,
stock_vs_spy_20d, price_vs_sma200, market_cap_m, signal_rank`.

### `market_characters`
`[{id, name}]` screens the ticker passes today.
