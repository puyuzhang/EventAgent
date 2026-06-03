# Historical Event Lab

Status: historical_event_lab v0.5 A-share MVP validated.

This isolated EventAgent submodule builds a historical event knowledge base for future real-time inference.
Global events remain the event inputs, but the main return observation target is now A-share / mainland China tradable proxies.

The current MVP covers:

- Gold-related events from 2020-01-01 to present
- Crude-oil-related events from 2020-01-01 to present
- Semiconductor-related events from 2020-01-01 to present
- Trump-related events from the current Trump term only, starting 2025-01-20

It does not claim complete historical coverage.

## Workflows

### Workflow A: Event Discovery

The runner loads curated discovery candidates from `config/event_candidates.json` and, when present,
the supplemental `config/event_candidates_expansion.json`, then:

1. Writes all raw candidates to `outputs/candidate_events_raw.json`.
2. Deduplicates similar candidates using first-trigger date, related asset, and title keywords.
3. Merges duplicate source links.
4. Writes deduped candidates to `outputs/candidate_events_deduped.json`.
5. Converts deduped candidates into asset-level event observations for return calculation.

Every candidate has source, URL, confidence, and `manual_check_required`.
Candidates may also define `asset_impacts[]`; this lets one parent event map to multiple
A-share proxy observations without changing the downstream return-calculation pipeline.

### Workflow B: A-Share Return Calculation

For each deduped event or asset impact:

1. Resolve the impact's related asset to `config/ashare_assets.json`.
2. Load A-share daily OHLCV prices from `data/raw_prices/`.
3. If no local file covers the event range and the proxy code is confirmed, try optional AKShare.
4. Do not fall back to GLD, USO, SOXX, Yahoo, Stooq, or yfinance for main A-share returns.
5. Entry date is the first China trading day strictly after `first_trigger.date`.
6. Entry price uses that bar's open price.
7. If open is missing and close exists, the close is used and the fallback is recorded.
8. Forward returns are calculated for 1, 3, 5, 10, 20, 60, and 120 trading days where enough bars exist.
9. Missing windows remain null and are reported.

## Major Event Methodology

A major event should satisfy at least one of:

- Reported by at least two credible sources
- Directly related to supply, demand, sanctions, tariffs, war/geopolitics, monetary policy, export controls, or major company/industry policy
- Associated with an abnormal price movement in the related asset
- Clearly recognized as market-moving in financial news or official statements

The MVP uses curated source links from official agencies and financial news. `manual_check_required` remains true so a human can review the event before production use.

## Run

From the EventAgent project root:

```powershell
python historical_event_lab/src/run_historical_event_lab.py
```

If Python is not on PATH in the Codex desktop environment, use the bundled Python executable reported by the workspace dependency loader:

```powershell
& 'C:\Users\23314\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe' historical_event_lab/src/run_historical_event_lab.py
```

The A-share runner now performs a preflight check before exporting:

- `config/ashare_assets.json` must have confirmed non-`TODO` proxy codes and exchanges.
- `data/raw_prices/ASHARE_GOLD_PROXY.csv`, `ASHARE_CRUDE_OIL_PROXY.csv`, and `ASHARE_SEMICONDUCTOR_PROXY.csv` must contain at least 1000 daily OHLCV rows.
- If a required A-share CSV is missing or empty, the runner tries AKShare first, then Eastmoney. If both fail, it stops with a clear error instead of exporting empty-price results.

To fetch or refresh A-share proxy CSVs explicitly:

```powershell
& 'C:\Users\23314\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe' historical_event_lab/src/download_ashare_prices.py --start 2020-01-01 --end 2026-06-03
```

To verify the output pipeline and print exact paths:

```powershell
& 'C:\Users\23314\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe' historical_event_lab/src/validate_ashare_pipeline.py
```

## A-Share Proxy Caveat

The crude-oil observation proxy is `501018.SH` 南方原油LOF. It is used as a China-tradable proxy for oil-related exposure, but it is not a pure crude-oil ETF. It has a QDII/FOF/LOF structure and may have NAV lag, premium/discount effects, trading suspension or quota behavior, and overseas-market timing mismatch. Returns for oil events should be read as A-share tradable proxy reactions, not direct crude-futures returns.

## Inputs

- `config/ashare_assets.json`: primary A-share proxy configuration, market scope, and return windows
- `config/historical_assets.json`: legacy US benchmark proxy configuration; not used by the A-share runner
- `config/event_candidates.json`: active curated discovery input
- `config/event_candidates_expansion.json`: supplemental curated expansion input with optional `asset_impacts[]`
- `config/seed_events.json`: deprecated placeholder, not used by the runner
- `data/raw_prices/*.csv`: optional audited local daily price files
- `data/raw_news/`: optional raw historical news files for future enrichment

Local A-share price files should use proxy-specific names such as:

```text
ASHARE_GOLD_PROXY.csv
ASHARE_CRUDE_OIL_PROXY.csv
ASHARE_SEMICONDUCTOR_PROXY.csv
```

Files should have English columns:

```text
date, open, high, low, close, volume
```

AKShare-style Chinese headers are also accepted:

```text
日期, 开盘, 最高, 最低, 收盘, 成交量
```

If `asset_code` is still `TODO` in `ashare_assets.json`, price loading is intentionally skipped and no US benchmark proxy is used.

## Outputs

- `outputs/candidate_events_raw.json`
- `outputs/candidate_events_deduped.json`
- `outputs/event_cases/*.json`
- `outputs/historical_event_knowledge_base.json`
- `outputs/historical_event_knowledge_base.csv`
- `outputs/price_data_audit_report.json`
- `outputs/price_data_audit_report.csv`
- `outputs/return_calculation_audit.csv`
- `outputs/signal_performance_summary.json`
- `outputs/signal_performance_summary.csv`
- `outputs/manual_review_queue.csv`
- `outputs/validation_summary.json`
- `outputs/run_report.json`

JSON is the primary knowledge base. CSV is only a flattened review/export format.

## Audit Outputs

`price_data_audit_report.json` and `price_data_audit_report.csv` audit the three A-share proxy price files. They report row coverage, duplicate dates, missing OHLCV fields, suspected price anomalies, data source, and whether each file passed the blocking data-quality checks.

`return_calculation_audit.csv` lists each event's trigger date, entry date, entry price, forward close prices, and stored forward returns. It is intended for manual verification of the return formulas without changing the primary event knowledge-base schema.

## Signal Evaluation

The flattened knowledge-base CSV includes `return_direction_*` and `signal_hit_*` columns for the configured return windows.

Signal-hit rules:

- `bullish` is a hit when the forward return is greater than zero.
- `bearish` is a hit when the forward return is less than zero.
- `uncertain` is excluded from hit-rate calculation and is stored as null in the signal-hit columns.
- Null forward returns produce null signal-hit values.

`signal_performance_summary.json` and `signal_performance_summary.csv` group results by `asset_code`, `asset_name`, `event_theme`, and `signal_direction`. The summary includes event counts, valid signal counts, actual hit counts, hit rates, average returns, median returns, and average 120-day max drawdown for the 1d, 5d, 20d, 60d, and 120d windows.

Signal summary count fields:

- `valid_signal_count_*`: non-null signal-hit denominator after excluding uncertain signals and null returns
- `hit_count_*`: number of correct bullish/bearish signals
- `hit_rate_*`: `hit_count_* / valid_signal_count_*`

## Manual Review Queue

`manual_review_queue.csv` ranks asset-level observations that need human review before production use. It includes `observation_id`, parent event IDs, source URL, mapped A-share asset, signal and mapping confidence, source reliability score, event importance score, review reason, and priority score.

Review priority increases when:

- `source_count` is 1
- one parent event maps to multiple assets
- absolute 20-day or 120-day returns are large
- `signal_direction` is uncertain but later returns are large
- the source URL or trigger date should be manually verified

`production_ready` is false by default while `manual_check_required` is true.

## Identifier Semantics

- `parent_event_id` is the unique event identifier before asset mapping.
- `observation_id` is the unique asset-level event-return observation identifier.
- One `parent_event_id` can map to multiple `observation_id` values.
- `asset_impact_id` is currently the same value as `observation_id`.
- `event_instance_id` is kept for backward compatibility and should not be treated as the unique asset-level key unless it is explicitly verified unique.

## JSON Schema

The knowledge base includes:

- `version`
- `created_at`
- `scope`
- `methodology`
- `events`

Each asset-level event observation includes:

- `event_id`, `observation_id`, `event_instance_id`, `parent_event_id`, `asset_impact_id`
- `event_importance_score`, `source_reliability_score`, `trigger_date_confidence`, `mapping_confidence`, `production_ready`
- `market_scope`, `event_theme`, `event_name`
- `first_trigger`: date, title, source, URL, summary, confidence, manual-check flag, and additional sources
- `asset`: asset id, name, region, market, code, exchange, data source, and proxy reason
- `asset_impact`: asset-impact id, parent event id, mapped asset id, impact summary, and mapping confidence
- `signal`: direction, reasoning, and confidence
- `price_data`: daily source, `first_china_trading_day_open_after_event` entry rule, entry date, entry price, and price-check flag
- `returns`: all configured windows plus 120-day max return and max drawdown
- `audit`: source count, source quality, selection criteria, news verification, price verification, merged candidate IDs, and notes

## CSV Schema

The CSV keeps the flattened schema:

```text
event_id,observation_id,event_instance_id,parent_event_id,asset_impact_id,market_scope,event_theme,event_name,first_trigger_date,first_trigger_title,first_trigger_source,first_trigger_url,asset_name,asset_region,asset_market,asset_code,exchange,entry_rule,signal_direction,signal_confidence,event_importance_score,source_reliability_score,trigger_date_confidence,mapping_confidence,production_ready,entry_date,entry_price,return_1d,return_3d,return_5d,return_10d,return_20d,return_60d,return_120d,return_direction_1d,return_direction_3d,return_direction_5d,return_direction_10d,return_direction_20d,return_direction_60d,return_direction_120d,signal_hit_1d,signal_hit_3d,signal_hit_5d,signal_hit_10d,signal_hit_20d,signal_hit_60d,signal_hit_120d,max_return_120d,max_drawdown_120d,news_verified,price_verified,manual_check_required,data_source,notes
```

## Return Formulas

For N trading days:

```text
return_Nd = close_at_entry_index_plus_N / entry_price - 1
```

`max_return_120d`:

```text
max(close_t / entry_price - 1) for closes from entry day through entry + 120 trading days
```

`max_drawdown_120d` starts with entry price as the initial peak:

```text
drawdown_t = close_t / running_peak_t - 1
```

The stored value is the minimum drawdown over the 120-trading-day window. If the full window is unavailable, 120-day max return and drawdown remain null.

## Validation

The run validates:

- Candidate outputs are written
- Event IDs are unique
- Required event fields are present
- Each event has `asset_region`, `asset_market`, and `asset_code`
- Every configured return window exists, even when null
- CSV columns match the expected flattened schema
- `run_report.json` is generated

## Limitations And Fixes

Known limitations:

- Coverage is curated and incomplete.
- The expanded event library is still a review set, not a complete event history.
- Several parent events map to multiple A-share proxy observations, so output row count is an
  asset-impact observation count rather than a unique parent-event count.
- Some source URLs, especially agency press pages, can move and should be manually checked.
- `501018.SH` 南方原油LOF is a QDII-FOF-LOF proxy and can diverge from direct crude-oil exposure because of NAV lag, premium/discount, and overseas-market timing.
- Recent events may have null 60-day or 120-day windows until enough future China trading days are available.
- The runner intentionally does not use US-listed benchmark ETFs for main A-share returns.

To fix missing prices:

1. Confirm the A-share proxy `asset_code` and `exchange` values in `config/ashare_assets.json`.
2. Add full audited daily OHLCV files named `ASHARE_GOLD_PROXY.csv`, `ASHARE_CRUDE_OIL_PROXY.csv`, and `ASHARE_SEMICONDUCTOR_PROXY.csv` under `data/raw_prices/`.
3. Or install `akshare` and run with network access enabled.
4. Re-run the lab and inspect `outputs/run_report.json`.
