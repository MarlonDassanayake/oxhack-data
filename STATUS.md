# Part 1 status

Contract version: 1.2.0  
Last updated: 2026-10-24

## Works

- Supplied v1.2.0 contract, integration guide, Part 1 specification, and change notes are installed unchanged. All 43 shared contract tests pass.
- Cached 21 season archives (168 league CSVs) cover E0, E1, SP1, D1, I1, F1, N1 and P1 from 2005-06 through 2025-26. Raw provider files stay in ignored `data/raw/` and are excluded from the handoff archive.
- Explicit team aliases map every observed source name. Unmapped teams fail with a review file rather than being guessed.
- Current real data build: 61,865 matches and outcomes; 630,288 quote rows across every available individual book and MAX/AVG aggregate, including invalid partial quotes; 619,442 valid market-probability rows; 1,632 efficiency rows. All eight leagues were previously hand-checked against source results.
- Bet365 is the reference in both snapshots. Outliers remain in the data and are audited by `quote_id`; this build flags 24,676. The audit records 345 COVID-extended 2019-20 fixtures, 41,332 kickoffs within two hours of midnight UTC, 64 harmless trailing-empty-field repairs, invalid quote counts, and source availability by file.
- The deterministic backtest calls the shared `reference_quote`, `best_price_quote`, and `decide_match` functions and writes all three policies: naive, controlled, and controlled-best-price. The synthetic validation run produced 1,314 rows for market, Elo and LightGBM. It is kept separate from the real data handoff.
- Ricky's public `rixzky/oxhack-models` main branch (commit `1360bdc9e3f2c5e350472e4cd64ea8a3751b7cf1`) was reviewed against contract v1.2.0. Its real-data runner produced and validated 117,880 real predictions for `market` and `elo` across warmup, train, calibration and validation; holdout was excluded. Ricky's forecast evaluator produced validation metrics on the 2,916 matches shared by both models: market log loss 0.972545 / Brier 0.579043; Elo 0.999040 / 0.597352.
- The real validation backtest produced 17,496 decision rows for `market` and `elo` across naive, controlled and controlled-best-price policies, covering 2,916 fixtures per model. No holdout decisions were run. The full Part 1 test suite passes (56 tests).
- A 12-file real validation handoff archive is available at `artifacts-real-validation.zip`; the synthetic ledger is kept out of that archive.
- Holdout metrics and decisions remain gated on the team's current-contract freeze.

## Waiting on team inputs

- Ricky's repository does not include generated artifacts (the `artifacts/` directory is gitignored). Real market/Elo predictions and validation decisions are now in this repo's `artifacts/`. The current repository contains Elo and Dixon-Coles code, but no `marketnet.py` or `lgbm.py`; those two planned model outputs remain pending.
- A full eight-league Dixon-Coles run was attempted from the current model source but was stopped during its first real-data fit after several minutes without producing output. It needs a faster/refined fitting run from Ricky before it can be included. The existing validated market/Elo outputs and decisions remain complete and usable.
- A holdout run needs the selected model/version and completed `frozen_config.json` from the team freeze.
- Optional Tiger Data loading is not implemented; there is no configured database connection.

## How to run

Use Python 3.11 from this repository root:

```sh
python -m pytest -q
python -m data_pipeline.download --leagues E0 E1 SP1 D1 I1 F1 N1 P1 --seasons 2005-06:2025-26
python -m data_pipeline.run --leagues E0 E1 SP1 D1 I1 F1 N1 P1 --seasons 2005-06:2025-26 --permission-note "Team owner confirmed intended hackathon training use."
python -m decisions.run_backtest --split validation --models market,elo,lgbm
python -m decisions.run_backtest --golden --split validation --models market,elo,lgbm
```

The real backtest reads `artifacts/predictions.parquet` and writes `artifacts/decisions.parquet` once Ricky shares Part 2's predictions. Do not run holdout before the freeze.

## Known assumptions and limits

- `QUESTIONS.md` records how v1.2.0 resolved all four cross-part questions. `Europe/London` remains a contract assumption for source kickoff times; the midnight audit is available in `artifacts/data_audit.json`.
- Source terms and the team's intended-use confirmation are recorded in `artifacts/data_manifest.md` and `artifacts/manifest.json`. This records the team's confirmation and does not create a provider licence. Raw CSVs are not redistributed.
- Quotes are sparse snapshots with `WINDOW` timestamps and differing bookmaker coverage. The outputs support leakage-aware evaluation; they do not establish profitable performance.
