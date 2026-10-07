---
name: alphasift
description: "Automated stock-screening Skill. Use when: the user wants to screen A-shares by strategy, list available strategies, run dual-low/volume-breakout/balanced-multifactor/capital-heat screens, or save runs and perform retrospective T+N evaluation. Return candidate stock lists through the alphasift CLI or Python API."
---

# alphasift — Automated stock-screening Skill

Screen, score, and rank A-share candidates by strategy.

Positioning: a full-market candidate-discovery and cross-candidate ranking engine. It sits upstream of in-depth single-stock analysis services such as `daily_stock_analysis`; DSA is an optional L3 post-analyzer rather than a main screening dependency.

## Use When

- The user wants to list currently available strategies.
- The user wants to screen A-shares with strategies such as `dual_low`, `volume_breakout`, `balanced_alpha`, and `capital_heat`.
- The user wants structured JSON results for subsequent agent analysis.
- The user wants to save screening runs and evaluate them later against the latest snapshot.

## Preconditions

The alphasift package must be installed in the current Python environment. If it is not installed, run:

```bash
pip install -e .
```

For LLM ranking, configure `LITELLM_MODEL`, `LLM_CHANNELS`, `LITELLM_CONFIG`, or the legacy variables `LLM_API_KEY/LLM_MODEL/LLM_BASE_URL`. You can directly reuse LiteLLM configuration fields from `daily_stock_analysis`, including `OPENAI_*`, `GEMINI_*`, `DEEPSEEK_API_KEY`, and `OLLAMA_API_BASE`. Strategy YAML can override default rules, event preferences, and candidate-context source weights through `scoring_profile`, `risk_profile`, `portfolio_profile`, `scorecard_profile`, and `event_profile`. The LLM outputs candidate sector/theme labels. If candidates provide `industry/concepts/board_heat_score/board_heat_trend_score`, these anchor the LLM, theme-heat factors, and portfolio diversification layer; a history sidecar can supply persistence, cooling, and state fields. The default diversification layer maps these labels to risk buckets to reduce repeated exposure to the same crowded trade.

L3 enables the local `scorecard` post-scorer by default; `dsa` or `external_http` can also be added. For DSA post-analysis, set `DSA_API_URL`; DSA provides post-processing enrichment only and does not participate in initial full-market screening.

Strategies requiring daily candlesticks automatically enrich the Top N candidates after L1.

For L3 in-depth analysis, set `DSA_API_URL`. Here, `DSA` refers to the external project `daily_stock_analysis`, with `POST /api/v1/analysis/analyze` as the default call.

## Operations

### 1. View available strategies

```bash
alphasift strategies
```

### 2. Run screening

```bash
alphasift screen dual_low --no-llm
alphasift screen balanced_alpha --no-llm
alphasift screen capital_heat
alphasift screen volume_breakout --max-output 10
alphasift screen balanced_alpha --context "Brokerage stocks are seeing higher volume today, with capital returning to undervalued financials"
alphasift --env-file /home/ubuntu/daily_ai_assistant/.env screen balanced_alpha
alphasift screen balanced_alpha --explain
alphasift screen balanced_alpha --candidate-context-file candidate_context.csv
alphasift screen dual_low --no-post-analysis
alphasift screen shrink_pullback --no-llm
alphasift screen dual_low --post-analyzer dsa
alphasift audit
alphasift industry-cache --output data/industry_map.csv --explain
alphasift screen dual_low --no-llm --save-run
alphasift runs
alphasift evaluate <run_id> --explain
alphasift evaluate-batch --limit 20 --explain
alphasift evaluate <run_id> --with-price-path --explain
```

### 3. Call through Python

```python
from alphasift import evaluate_saved_run, evaluate_saved_runs, screen, list_strategies

list_strategies()
screen("dual_low", market="cn", use_llm=False)
evaluate_saved_run("<run_id>")
evaluate_saved_runs(limit=20)
```

## Output

Returns `ScreenResult` JSON with the following core fields:
- `strategy`: Strategy name
- `market`: Market
- `strategy_version`: Strategy version
- `snapshot_count`: Number of stocks in the full-market snapshot
- `after_filter_count`: Number remaining after hard filtering
- `picks`: Candidate list
- `llm_ranked`: Whether LLM ranking was applied
- `llm_market_view`: LLM's overall assessment of the candidate pool and market environment
- `llm_selection_logic`: Core judgment dimensions used by the LLM for this ranking
- `llm_portfolio_risk`: Shared risks identified by the LLM in the final list
- `llm_coverage`: Fraction of the candidate pool covered by LLM output
- `post_analyzers`: Enabled L3 post-analyzers
- `daily_enriched`: Whether candidate daily candlestick enrichment was applied
- `risk_enabled`: Whether the independent risk layer is enabled
- `portfolio_concentration_notes`: Penalty explanations from the portfolio diversification overlay
- `degradation`: Degradation information
- `snapshot_source`: Data source actually used
- `source_errors`: Errors from data sources that failed before fallback

Each `Pick` contains `factor_scores`, `industry/concepts/board_heat_score/board_heat_trend_score/board_heat_persistence_score/board_heat_cooling_score/board_heat_state/board_heat_summary`, LLM thesis/reasons/risks/catalysts/sectors/themes/tags/style fit/watch items/invalidators, risk-layer fields, portfolio diversification penalties, and optional post-analysis fields. DSA fields are populated only when the `dsa` analyzer is enabled.

Each `Pick` may also contain:
- `deep_analysis_status`
- `deep_analysis_summary`
- `deep_analysis_result`
- `deep_analysis_signal_score`
- `deep_analysis_sentiment_score`
- `deep_analysis_operation_advice`
- `deep_analysis_trend_prediction`
- `deep_analysis_risk_flags`

## Boundaries

- Currently supports only `market="cn"`.
- There is currently no standalone remote `get_result` service; manage run records locally with `--save-run`, `runs`, `evaluate`, and `evaluate-batch`.
- `audit` checks strategy profile coverage, known capability gaps, and next-step priorities.
- `--candidate-context-file` supports CSV/JSON/JSONL and aligns candidate-level news, announcements, fund flows, or research summaries by `code`, injecting only rows relevant to the current candidate pool. Optional fetching includes `source_count`, `source_confidence`, `source_weight_score`, `context_summary`, and announcement categories.
- Candidate-level context identifies broad event and negative-risk tags for cross-candidate LLM ranking.
- `industry-cache` caches industry/concept mappings and sector-heat fields and writes a history sidecar. Later mapping loads can supply rolling sector-heat trends, persistence, cooling, and state fields for LLM context and the `theme_heat` factor.
- The diversification layer prefers LLM sector/theme labels and can fall back to the candidate's `industry` field; if both are missing, it does not change the rule-based score.
- L3 post-analyzers run only on final candidates and do not participate in initial full-market screening. The local `scorecard` is enabled by default; DSA is one optional additional backend.
- T+N evaluation uses saved prices and the latest snapshot prices at evaluation time, rather than a full backtest using adjusted prices. It can deduct transaction costs and output retrospective breakout/pullback labels; `--with-price-path` also estimates maximum drawdown and maximum favorable excursion.
