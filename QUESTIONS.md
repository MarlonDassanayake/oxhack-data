# Questions for the contract owner

All four questions from the v1.1.0 handoff were answered in contract v1.2.0. No open contract questions remain for Part 1.

| Question | v1.2.0 resolution | Part 1 handling |
|---|---|---|
| 2019-20 source-season dates | Source season is authoritative; `contract.schemas.season_date_bounds` extends 2019-20 through 2020-08-31. | Keep those fixtures in 2019-20 and record the exception count in the manifest and audit. This build retained 345. |
| Pre-closing quote timing | Decision time is the information cutoff; execution occurs later at the pre-closing price. Store only the conservative capture-window upper bound and reject quotes at or after kickoff. | Quotes remain `WINDOW`; `contract.decisions.decide_match` enforces the kickoff boundary. |
| Bet365, Pinnacle, execution and CLV | Bet365 is the reference in every season. Compare naive, controlled, and controlled-best-price policies. Exclude Pinnacle from 2025-26 execution; always measure CLV against Bet365 closing odds. | Build Bet365-only references, preserve PS for analysis, and pass audit outlier IDs to the shared best-price function. |
| Kickoff timezone | Retain `Europe/London` as the shared contract assumption and audit kickoffs within two hours of midnight UTC. | Convert source times as required and report the audit count; do not present the timezone as provider-confirmed. |

