# SPEC: Part 1, Data and Decisions (`oxhack-data`)

Read `CONTRACT.md` first. It wins over this file. You work in your own repo; you hand
off **artifact files**, not code (see `INTEGRATION.md`). You are the foundation for the
other two parts, so correctness and on-time hand-offs matter more than breadth.

## 1. Mission

1. Turn football-data.co.uk CSVs into clean, validated, leakage-safe tables; compute
   margin-free market probabilities; produce the **efficiency map** (the hero finding);
   document exactly what the data can and cannot support.
2. Run the **backtest**: turn Part 2's predictions into a decisions ledger using the
   shared single-match function `contract.decisions.decide_match`, plus batching,
   bankroll, exposure caps and settlement (CONTRACT 6.3).

Data work front-loads (most of it done by Sat evening). The backtest is deterministic
accounting with exact test vectors; start it on golden files early so it is ready when
real predictions arrive.

## 2. You produce (into `artifacts/`, validated with `contract.schemas.write_table`)

| File | Notes |
|---|---|
| `matches.parquet` | one row per match, canonical names, `feature_cutoff_utc` and `matchweek_key` via contract helpers |
| `outcomes.parquet` | goals and `result` only |
| `quotes.parquet` | one row per match x bookmaker x snapshot for EVERY individual bookmaker present (B365, PS, BW, IW, WH, VC) plus aggregates (MAX, AVG); invalid rows kept with `is_valid=False` and `invalid_reason` |
| `market_probs.parquet` | from every valid quote; `is_reference=True` on the Bet365 row (one per match x snapshot) |
| `efficiency_map.parquet` | league x season x snapshot x bookmaker metrics (section 5); excludes 2025-26 until the freeze |
| `manifest.json` | `dataset_mode: "REAL"`, contract_version, leagues, seasons, sources with terms check and download date |
| `splits.json` | `by_season` role for every season (CONTRACT 8), `holdout_season: "2025-26"` |
| `data_audit.json` | shape of `contract/golden/artifacts_json/data_audit.json` |
| `data_manifest.md` | provenance and limitations (section 8) |
| `decisions.parquet` | the ledger: every match x model x policy, BET and NO_BET with reasons (section 6) |
| `frozen_config.json` | at the freeze only (CONTRACT 8) |
| `trials_log.jsonl` | append one line per policy setting you try on validation (`"part": 1`) |

Code: `data_pipeline/` (`run.py` entry point, `team_aliases.csv`, `load_tiger.py`) and
`decisions/` (`run_backtest.py`, `bankroll.py`, `freeze.py`, `run_holdout.py`), `tests/`.

## 3. Priorities

**P0, by Sat 16:00:** E0 data slice end to end (download, parse, canonical names,
matches, outcomes, quotes for every bookmaker present, PRE_CLOSING and CLOSING,
market_probs with Bet365 as reference, splits, manifest, audit). All validate. Post `artifacts.zip`.
Backtest skeleton running on golden files and passing the `bankroll` vector.
**P0, by Sat 22:00:** all leagues (E0, E1, SP1, D1, I1, F1, N1, P1); efficiency map;
first real `decisions.parquet` on validation for whatever models Part 2 has posted.
**P0, by Sun 03:00:** backtest re-run for all models; policy trials logged.
**P1:** Tiger Data loader and the replay query (`contract/tiger_schema.sql`), tested
against a real database if credentials exist. Quarter-Kelly runs (after flat works).
**P2:** AVG bookmaker rows for the efficiency map. Understat xG only if Part 2 asks.

## 4. Ingestion and cleaning rules

- Source URL pattern (verify): `https://www.football-data.co.uk/mmz4281/{YYZZ}/{LEAGUE}.csv`,
  e.g. `2324/E0.csv` for 2023-24. Save raw files to `data/raw/` (gitignored). Read
  `notes.txt` and the site's terms first; write findings into `data_manifest.md` and
  report to the team by Sat 14:00 (CONTRACT section 14).
- Columns (verify names per season): `Date`, `Time`, `HomeTeam`, `AwayTeam`, `FTHG`,
  `FTAG`, `FTR`; pre-closing `{BK}H {BK}D {BK}A` and closing `{BK}CH {BK}CD {BK}CA` for
  each individual bookmaker prefix present (`B365`, `PS`, `BW`, `IW`, `WH`, `VC`; older
  files may use `PH PD PA` for Pinnacle); aggregates `MaxH/AvgH` (and closing
  `MaxCH/AvgCH`), older `BbMxH/BbAvH`, stored as `MAX`/`AVG`. Never guess a column
  you have not seen; record per-season bookmaker coverage in the manifest.
- Season: the source file's season. Dates `dd/mm/yy` and `dd/mm/yyyy`, `dayfirst=True`;
  check each date against `contract.schemas.season_date_bounds(season)` (2019-20 runs to
  2020-08-31); count the 2019-20 exception rows in the manifest; reject other
  out-of-bounds rows.
- Kickoff: `Time` present -> `Europe/London` to UTC, `EXACT`. Absent -> 00:00 UTC that
  date, `DATE`. This timezone is a documented assumption (CONTRACT 4.1); report in the
  audit how many kick-offs fall within 2 hours of midnight UTC.
- Quote timestamps: CONTRACT 4.1. PRE_CLOSING `observed_at_utc` is the LATEST plausible
  capture time of the documented window (conservative). Never invent an exact time.
- Team names: `team_aliases.csv` (`raw_name,canonical_name`), one canonical name per
  club across seasons. Fail loudly on unmapped names.
- Duplicates: same league, date and teams -> keep first, count in audit.
- Invalid quotes: any odd missing, non-finite or <= 1.0 -> `is_valid=False`,
  `invalid_reason` in {`MISSING_ODDS`, `ODDS_LE_1`, `NON_FINITE`}.
- Outliers (flag, do NOT drop): overround < 0 or > 0.25; any odd > 100; a bookmaker's
  pre-closing q differs from Bet365's q by > 0.15 on any outcome. List them in
  `data_audit.json` `flagged_outliers` by `quote_id`; these become `is_outlier=True`
  when building best-price quotes (yours and the UI's).
- Exclusions (dropping whole matches) are decided without looking at outcomes and
  are listed with reasons.
- Holdout season 2025-26 is ingested and validated like any other season.

## 5. Market probabilities and the efficiency map

- `market_probs`: `contract.maths.implied_probs` per valid quote; `is_reference=True`
  on the Bet365 row (`schemas.REFERENCE_BOOKMAKER`), separately for PRE_CLOSING and
  CLOSING.
- `efficiency_map`, per league x season x snapshot x bookmaker with at least 100 valid
  matches: `log_loss`, `brier` (sum convention), `mean_overround`; `calib_slope`
  (logistic regression of the outcome indicator on `logit(q)`, pooled over outcomes;
  1 = calibrated, below 1 = overconfident); `fav_longshot_gap` = mean(y - q) where
  q < 0.25 minus mean(y - q) where q > 0.6 (negative = classic favourite-longshot
  bias); `draw_bias` = mean(y_draw - q_draw). Write the sign conventions into
  `data_manifest.md`.
- Leave 2025-26 out until the freeze, then regenerate including it.

## 6. Backtest (the `decisions/` package)

Inputs: your own tables plus Part 2's `predictions.parquet`.
For each model and each policy in `contract.decisions.POLICIES` (`naive`,
`controlled`, `controlled_best_price`; at the freeze the frozen policy replaces the
`controlled` settings and `controlled_best_price` uses the same settings with
`price_source = BEST_AVAILABLE`), over one split:

1. For every match, collect its PRE_CLOSING quotes as dicts (keys in the
   `best_price_quote` docstring, `is_outlier` from your audit flags) and get the model's
   prediction. Executable quote: `reference_quote(quotes)` for `price_source =
   REFERENCE`; `best_price_quote(quotes, season, policy["best_price_max_prob_gap"])`
   for `BEST_AVAILABLE` (fall back to `reference_quote` if it returns None, which then
   fails as INVALID_PRICE or MISSING_QUOTE).
2. Process batches in time order (batch = date of `decision_time_utc`). Before each
   batch, settle every open bet whose kickoff date + 1 day <= the batch date.
   `available = settled bankroll - open stakes`.
3. Call `decide_match(probs, quote, kickoff_utc, policy, available_bankroll=available)`
   for each match in the batch. Do not reimplement any of its logic.
4. Sort the batch by `ev_stressed` desc, then `match_id`. Place `BET`s until total
   open stakes would exceed `max_open_exposure_frac * settled`; later ones get
   `EXPOSURE_LIMIT_TOTAL` (stake 0, explanation from `contract.decisions.explain`
   messages). Stake above available -> `INSUFFICIENT_BANKROLL`.
5. Settle at the executed bookmaker's OFFERED odds and store that bookmaker in the
   `bookmaker` column. If a Bet365 closing quote exists, fill `closing_fair_prob` (same
   outcome, Bet365 closing margin-free) and `clv` with `contract.maths.clv`.
6. Write rows with `decision_id = f"{run_id}-{policy_id}-{model_id}-{match_id}"`,
   `config_hash` of the policy, `is_exploratory=False`.

`contract/golden/generate_golden.py` contains a short golden sketch of this loop; use
it as a reference for shapes, but write and test your own in `decisions/`.

## 7. Tests you must write (`tests/`)

- Contract tests pass (`python -m pytest contract -q`).
- Parsing: fixture CSV with both date formats gives the right UTC times.
- Team aliases: every raw name maps; unmapped name raises.
- Reference rule: Bet365 is the reference in every season; no reference when Bet365 is
  missing or invalid.
- Best-price quotes reproduce the `best_price` vectors; Pinnacle is never executed in
  2025-26; MAX/AVG never executed.
- `feature_cutoff_utc == contract.schemas.feature_cutoff(kickoff_utc)` for all rows.
- One hand-checked match per league: odds and result equal the raw CSV.
- Exclusion logic does not read `outcomes`.
- Backtest: `bankroll` and `quarter_kelly_stake` vectors in
  `contract/golden/test_vectors.json`; unsettled winnings never reused; a bet from an
  earlier batch still open counts against the next batch's exposure; settlement uses
  offered odds; NO_BET rows are written too.

## 8. `data_manifest.md` must state

Source and URLs, download date, terms checked (who, when, what they say), leagues and
seasons, per-season availability of PS, B365 and closing columns, timestamp precision
assumptions and evidence, known limitations (no opening odds, approximate pre-closing
capture time, closing only for some seasons), audit counts, sign conventions.

## 9. Do not

- Do not edit `contract/`. Ask the contract owner.
- Do not commit raw football-data files or `artifacts/`.
- Do not call any snapshot "opening". Do not fill missing odds.
- Do not drop rows because of their outcome.
- Do not compute or backtest anything on 2025-26 before the freeze (data checks only).
- Do not compute ROI, Sharpe, bootstrap or other policy metrics (Part 2 does). Do not
  build models or UI.

## 10. Hand-off checklist (every time you post `artifacts.zip`)

- [ ] All parquet files pass `read_table`.
- [ ] `manifest.json` has the right `dataset_mode`, leagues and seasons.
- [ ] Audit counts are consistent (raw >= accepted).
- [ ] `decisions.parquet` covers every match x model x policy for the split.
- [ ] Short channel message: what changed, coverage, known issues.
