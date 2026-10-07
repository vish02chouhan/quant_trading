# Usage guide

This document covers everyday usage beyond the README: installation, CLI commands, Python calls, context injection, and the evaluation loop.

## Installation

```bash
pip install -e .
cp .env.example .env
```

If you do not need LLM ranking yet, add `--no-llm` when running commands; no model key is required.

## Run through the workflow in three steps

```bash
alphasift strategies

alphasift screen dual_low --no-llm --explain

alphasift screen dual_low --no-llm --save-run
alphasift runs
alphasift evaluate <run_id> --explain
```

## UI/agent overview

`overview` combines strategy groups, selection facets, strategy cards, strategy recommendations, source health, freshness/cache status, strategy field coverage, source history, strategy performance history, recent runs, and next actions in one payload for the first screen of a Web UI, notification assistant, or agent:

```bash
alphasift overview --explain
alphasift overview --risk-profile aggressive --data-requirement daily_k --match-limit 2 --json
alphasift serve --host 127.0.0.1 --port 8765
```

By default, it makes no network requests and only reads the current process's source health and local run index. Add `--live-data-check` for a real source smoke check.

`alphasift serve` starts a read-only local JSON API so UIs, agents, or external orchestration layers can consume stable payloads directly. It listens on `127.0.0.1:8765` by default. Available endpoints include `/health`, `/result-schema`, `/overview`, `/strategies`, `/strategy?name=<strategy_name>`, `/strategy-compare?base=<base>&target=<target>`, `/strategy-facets`, `/strategy-cards`, `/strategy-readiness`, `/strategy-run-summary`, `/data-source-history`, `/strategy-performance`, `/strategy-templates`, `/strategy-template?name=<template_name>`, `/runs`, `/report?run=<run_id>`, and `/doctor/data-sources`. The HTTP API also skips live source checks by default; use `/overview?live=true`, `/strategy-cards?live=true`, `/strategy-readiness?live=true`, or `/doctor/data-sources?live=true` when needed.

The strategy catalog can output more complete capability descriptions to help UIs, agents, or external systems select suitable strategies:

```bash
alphasift strategies --explain
alphasift strategies --json
curl "http://127.0.0.1:8765/strategy-facets"
curl "http://127.0.0.1:8765/strategy-cards?strategy=dual_low"
curl "http://127.0.0.1:8765/data-source-history?limit=50"
curl "http://127.0.0.1:8765/strategy-performance?limit=50"
```

Structured output includes strategy categories, tags, styles, data dependencies, daily-data requirements, required snapshot/daily fields, active hard filters, factor weights, and profile overrides. `/strategy-facets` summarizes these into `value/count/strategies` lists that can drive filtering controls directly, and identifies the corresponding query parameters. `/strategy-cards` also outputs `lanes` grouping strategies into `needs_history`, `needs_evaluation`, `performance_leaders`, and `attention`, allowing a UI to render sections for strategies awaiting runs, awaiting evaluation, leading in performance, or needing attention.

Strategies can also be matched directly by style/data dependency for CLI, Web UI, or agent selection:

```bash
alphasift strategies --risk-profile defensive --holding-period swing --market-regime risk_off --strict --explain
alphasift strategies --risk-profile aggressive --data-requirement daily_k --limit 2 --json
```

Matches contain `score`, `matched`, and `missing`, so interfaces can explain recommendations and show unmet preferences.

Before iterating on or adding strategies, compare their styles, data dependencies, required fields, hard-filter parameters, and factor weights:

```bash
alphasift strategies --compare dual_low low_volatility_quality --explain
alphasift strategies --compare dual_low low_volatility_quality --json
curl "http://127.0.0.1:8765/strategy?name=low_volatility_quality"
curl "http://127.0.0.1:8765/strategy-compare?base=dual_low&target=low_volatility_quality"
```

JSON output includes `differences` and `summary.compatibility_notes` for displaying parameter changes, daily-data dependency changes, and source-compatibility impacts. The HTTP API returns the same comparison structure, allowing a frontend to open a diff review from strategy details.

When adding a strategy, list templates first, then write template YAML under `strategies/`, rename it, and iterate:

```bash
alphasift strategies --templates --explain
alphasift strategies --template defensive_value_quality > strategies/my_defensive_value_quality.yaml
alphasift strategies --template momentum_breakout_daily --json
curl "http://127.0.0.1:8765/strategy-template?name=momentum_breakout_daily"
```

Template payloads contain strategy styles, data dependencies, applicability notes, and editable YAML, helping UIs/agents generate drafts without mixing templates into the enabled-strategy catalog.

Precheck source field coverage for a specific strategy:

```bash
alphasift doctor data-sources --strategy low_volatility_quality --no-live --explain
alphasift doctor data-sources --all-strategies --no-live --explain
alphasift doctor data-sources --strategy dual_low --compare-snapshot-sources --explain
curl "http://127.0.0.1:8765/strategy-readiness"
```

Single-strategy mode lists required snapshot and daily-feature fields. All-strategy mode outputs a coverage matrix and `strategy_readiness_summary`, helping UIs/APIs or agents determine which strategies are usable, which lack live checks, and which fields are critical to source stability. Removing `--no-live` performs a real fetch smoke test and outputs `snapshot_missing`, `daily_missing`, `source_errors`, `freshness_summary`, and repair suggestions for missing fields, expired caches, or source degradation. `--compare-snapshot-sources` checks each configured snapshot provider and compares row counts, required-field coverage, field quality, failed sources, and stock-code intersections, exposing sources that work but lack essential strategy fields.

JSON output also includes raw `source_health` counters, aggregated `health_summary`, `freshness_summary`, and the live snapshot's `quality_summary`. `health_summary` groups sources into `healthy_sources`, `failing_sources`, `disabled_sources`, and `never_seen_sources` for displaying health, circuit-breaker state, and recent errors. `snapshot.quality_summary` counts duplicate codes, field missingness, invalid numbers, and nonpositive prices/trading values/market caps, preventing apparently successful row counts from hiding unusable field quality.

## Common scenarios

Use cross-candidate LLM ranking:

```bash
alphasift screen balanced_alpha
```

Reuse another project's LiteLLM configuration file:

```bash
alphasift --env-file /home/ubuntu/daily_ai_assistant/.env screen balanced_alpha
```

LLM ranking with market, theme, or news context:

```bash
alphasift screen balanced_alpha --context "Brokerage stocks are seeing higher volume today, with capital returning to undervalued financials"
```

Inject news, announcements, fund flows, or research summaries aligned by candidate code:

```bash
alphasift screen balanced_alpha --candidate-context-file candidate_context.csv
```

Run the local L3 scorecard post-scorer by default:

```bash
alphasift screen balanced_alpha --explain
```

Add DSA as one optional L3 post-analyzer:

```bash
alphasift screen dual_low --post-analyzer dsa
```

Explicitly disable L3 post-scoring or analysis:

```bash
alphasift screen dual_low --no-post-analysis
```

Project and strategy self-check:

```bash
alphasift audit
alphasift audit --json
```

Refresh the industry, concept, and sector-heat mapping cache:

```bash
alphasift industry-cache --output data/industry_map.csv --explain
alphasift screen balanced_alpha --industry-map-file data/industry_map.csv
```

Batch-evaluate recently saved runs:

```bash
alphasift evaluate-batch --limit 20 --explain
alphasift evaluate-batch --limit 20 --with-price-path --failure-samples 10 --json
```

Batch evaluation outputs `failure_review`, aggregating samples with negative returns, missing quotes, failed breakouts, or severe drawdowns by strategy, LLM catalysts/risks, post-analysis tags, merged event signals, risk flags, patterns, and failure reasons, with next-step tuning or data-check suggestions. It also outputs `event_signal_review`, calculating win rates and returns for `tag:`, `catalyst:`, `risk:`, and `post:` event signals, with `prefer` / `avoid` / `watch` recommendations.

Generate a review report for a saved run:

```bash
alphasift runs --json
alphasift runs --strategy low_volatility_quality --json
alphasift performance --limit 50 --explain
alphasift report <run_id> --output data/reports/<run_id>.md
alphasift report <run_id> --json --output data/reports/<run_id>.json
curl "http://127.0.0.1:8765/strategy-run-summary?limit=50"
```

`runs --json` outputs a lightweight run index containing strategy version, category, source, LLM/daily status, degradation counts, a few error/degradation examples, and suggested report paths. `/strategy-run-summary` aggregates these indexes by strategy, producing run counts, latest reports, total candidates, source-error/degradation counts and examples, LLM/daily coverage, and recent-run cards without live quotes. `/data-source-history` aggregates recent runs by snapshot source, producing error/degradation rates, error/degradation examples, last-good fallback counts, stability states/scores, strategy coverage, watchlists, and next actions for panels monitoring recurring source failures and their reasons. `performance` / `/strategy-performance` reads saved evaluation files and outputs retrospective returns, win rates, performance scores, outcomes, leaderboards, and next actions by strategy without refetching quotes. Reports default to Markdown for human reviews, notifications, or daily summaries; `report --json` outputs a stable `RunReport` payload for Web UIs, agents, or external services. Add `--evaluate` to include the latest T+N evaluation summary.

Fetch daily price paths during evaluation to output maximum drawdown and maximum favorable excursion:

```bash
alphasift evaluate <run_id> --with-price-path --explain
```

## Python API

```python
from alphasift import evaluate_saved_run, evaluate_saved_runs, screen

result = screen("dual_low", use_llm=False)
for p in result.picks:
    print(f"{p.rank}. {p.code} {p.name} score={p.final_score:.1f}")
```

## Saving and evaluating

`alphasift screen --save-run` saves strategy version, sources, degradation records, candidates, scores, risk fields, post-analysis results, and prices at saving. Later, use:

```bash
alphasift evaluate <run_id> --explain
alphasift evaluate-batch --limit 20 --explain
alphasift runs --strategy dual_low --json
```

Evaluation uses saved prices and the latest snapshot prices at evaluation time to calculate T+N returns, win rates, missing quotes, transaction-cost deductions, equal-weight portfolio summaries, and retrospective pattern labels. `--with-price-path` also estimates maximum drawdown and maximum favorable excursion.
`evaluate-batch` / `evaluate-strategies` additionally outputs `failure_review` and `event_signal_review` to identify repeatedly failing strategies, event signals, risk flags, pattern states, and data issues, and derive event tags to prefer/avoid. `event_signal_review.strategy_patch_suggestions` converts strategy-level event win rates into reviewable `screening.event_profile` YAML snippets so stable prefer/avoid conclusions can later be applied to strategies.

## Custom strategies

Add YAML files under `strategies/`; the filename is the strategy identifier. Use `alphasift strategies --templates --explain` to view starting templates. See [strategy-guide.md](strategy-guide.md) for the full format and [../strategies/README.md](../strategies/README.md) for built-in descriptions.
