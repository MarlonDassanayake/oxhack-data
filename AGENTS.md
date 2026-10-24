# AGENTS.md: Part 1, Data and Decisions (repo `oxhack-data`)

Instructions for the coding agent (Codex reads this file; Claude Code reads it through
`CLAUDE.md`). You are one of three agents. Each works in a separate repo for one person
on a 3-person hackathon team. The parts are combined at the end by copying folders and
exchanging files, so following the shared contract exactly is the most important rule.

## Read first, in this order
1. `CONTRACT.md`: shared rules, schemas, formulas, timeline. It overrides everything.
2. `SPEC.md`: what THIS part must build, in priority order.
3. `INTEGRATION.md`: how parts hand off and how the final merge works.
4. `contract/`: shared code (`schemas.py`, `maths.py`), golden files, test vectors.

## Your part
- Produces: matches, outcomes, quotes, market_probs, efficiency_map, decisions (parquet) plus manifest.json, splits.json, data_audit.json, data_manifest.md, frozen_config.json (at the freeze).
- Consumes: raw football-data.co.uk CSVs, and Part 2's predictions.parquet for the backtest (golden copies in contract/golden/ until real ones arrive).
- Your code lives in `data_pipeline/`, `decisions/`. Tests live in `tests/`.
- First task: Two tracks. (a) Build the E0 data slice end to end: download, parse, team aliases, matches, outcomes, quotes, market_probs, splits, manifest, audit, all passing validators; post artifacts.zip by Sat 16:00. (b) In parallel, build the backtest in decisions/ on the golden tables using contract.decisions.decide_match until the bankroll vectors pass.

## Hard rules
1. Never edit `contract/`, `CONTRACT.md` or `INTEGRATION.md`. If they are wrong or
   silent on something that affects another part, write the question in
   `QUESTIONS.md` (what, why, proposed answer), tell the human, and continue with
   work that does not depend on it.
2. Write every shared table with `contract.schemas.write_table` and read with
   `read_table`. Exact file names and columns from CONTRACT section 10. Extra columns
   must start with `x_`.
3. Use `contract.maths` for implied probabilities, EV, stress, adverse odds, Kelly,
   log loss, Brier and CLV, and `contract.decisions.decide_match` for any single-match
   decision. Never reimplement them.
4. Never import code from the other two parts. Exchange only artifact files.
5. Respect the time and leakage rules (CONTRACT section 5) and the holdout protocol
   (section 8). Never use closing odds or results as inputs to a prediction or
   decision.
6. Honesty (section 9): never invent data, timestamps, results or integrations;
   carry `dataset_mode`; label synthetic and exploratory outputs.
7. The project has no name yet. Do not invent one anywhere (code, UI, docs).
8. Stay in scope: nothing from the CONTRACT cut list, nothing owned by another part.
9. Read `ARTIFACTS_DIR` (default `./artifacts`); never hardcode absolute paths.
   Run modules with `python -m` from the repo root. Python 3.11.
10. Do not commit secrets, raw data or `artifacts/`.

## Commands
    pip install -r requirements.txt
    python -m pytest -q            # contract tests + part tests, must pass
    python -m data_pipeline.run --leagues E0 --seasons 2005-06:2025-26
    python -m decisions.run_backtest --golden --split validation --models market,elo,lgbm
    python -m data_pipeline.load_tiger   # optional, needs DATABASE_URL

## Working style
- Build the smallest end-to-end slice on the golden files first, then deepen.
- Small commits with clear messages. Run the tests before saying something works.
- Keep `STATUS.md` current at every milestone: what works, what is stubbed, how to run
  it, known issues, contract version used. The final merge relies on it.
- When unsure between two readings of the spec, pick the more conservative one
  (less leakage, fewer claims) and note it in `STATUS.md`.

## Definition of done (for any task)
- `python -m pytest -q` passes, including `contract/test_contract.py`.
- Outputs validate against the contract and follow the exact file names.
- `STATUS.md` updated.
