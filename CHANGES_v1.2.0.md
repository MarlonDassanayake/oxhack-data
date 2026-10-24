# Contract v1.2.0 update (answers Q1 to Q4)

Everyone: replace your `contract/` folder, `CONTRACT.md` and `INTEGRATION.md` with the
ones in this update, replace your `SPEC.md` with your part's new one, then run
`python -m pytest -q` (43 contract tests should pass). Do not edit `contract/` locally.

## Decisions
- **Q1, 2019-20 boundary:** the source file's season wins. `schemas.season_date_bounds`
  allows 2019-20 to run to 2020-08-31. Count those rows in the manifest.
- **Q2, quote time vs decision time:** `decision_time_utc` is the information time;
  the bet executes later at the pre-closing quote, which is conservative, not leakage.
  PRE_CLOSING `observed_at_utc` now stores the LATEST plausible capture time, and
  `decide_match` rejects any quote observed at or after kickoff (`STALE_QUOTE`).
- **Q3, Pinnacle and bookmakers:** Bet365 is the reference market for every season
  (benchmark, model inputs, `naive` and `controlled` execution, CLV). New third policy
  `controlled_best_price` executes at the best valid price across individual
  bookmakers via `contract.decisions.best_price_quote` (no MAX/AVG aggregates, no
  flagged outliers, no Pinnacle in 2025-26, stale-price guard). Part 2 reports the
  line-shopping value with a bootstrap interval.
- **Q4, timezone:** keep `Europe/London`, documented as an assumption; Part 1 reports
  how many kick-offs fall within 2 hours of midnight UTC.

## What each part changes
- **Part 1, Data and Decisions:** ingest every individual bookmaker present plus
  MAX/AVG; reference = Bet365; latest-capture WINDOW times; season bounds helper;
  midnight count in the audit; backtest runs all three policies, builds quotes with
  `reference_quote` / `best_price_quote`, writes the new `bookmaker` column, CLV vs
  Bet365 closing.
- **Part 2, Models and Evaluation:** market features are Bet365 everywhere (re-run
  anything built on Pinnacle reference rows); policy metrics for three policies plus
  `line_shopping` in each `policy_metrics.json` row.
- **Part 3, App:** replay gets a policy/price-source toggle and shows the bookmaker per
  outcome; evidence shows three policies and the line-shopping value with its caveat.
  Updated types in `contract/api_types.ts` and examples in `contract/golden/api/`.

## Schema changes
- `decisions`: new nullable column `bookmaker`; `policy_id` may be `controlled_best_price`.
- `BOOKMAKERS` enum: B365, PS, BW, IW, WH, VC, MAX, AVG.
- New in `contract/schemas.py`: `REFERENCE_BOOKMAKER`, `AGGREGATE_BOOKMAKERS`,
  `EXECUTION_EXCLUDE`, `season_date_bounds`.
- New in `contract/decisions.py`: `best_price_quote`, `reference_quote`,
  `BEST_PRICE_POLICY`, `POLICIES`; `decide_match` returns `bookmaker`.
