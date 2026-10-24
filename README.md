# oxhack-data: Part 1, Data and Decisions

One of three repos for our OxHack '26 project (name to be decided). See `CONTRACT.md`
for the shared plan and rules, `SPEC.md` for this part, `INTEGRATION.md` for hand-offs.

## Setup
```
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m pytest -q
```

## Run
```
python -m data_pipeline.run --leagues E0 --seasons 2005-06:2025-26
python -m decisions.run_backtest --golden --split validation --models market,elo,lgbm
python -m data_pipeline.load_tiger   # optional, needs DATABASE_URL
```

## Getting started with an AI agent
Open this folder in Claude Code or Codex and say:
"Read AGENTS.md, CONTRACT.md and SPEC.md, then give me a short plan for the first
milestone and start building it."
