---
name: alphasift
description: "Automated stock-screening Skill. Use when: the user wants to screen A-shares by strategy, list available strategies, run dual-low/volume-breakout screens, or save runs and perform retrospective T+N evaluation. Return candidate stock lists through the alphasift CLI or Python API."
---

# alphasift — Automated stock-screening Skill

Screen, score, and rank A-share candidates by strategy.

Positioning: a full-market candidate-discovery and cross-candidate ranking engine. It sits upstream of in-depth single-stock analysis services such as `daily_stock_analysis`; DSA is an optional L3 post-analyzer rather than a main screening dependency.

## Use When

- The user wants to list currently available strategies.
- The user wants to screen A-shares with strategies such as `dual_low` and `volume_breakout`.
- The user wants structured JSON results for subsequent agent analysis.
- The user wants to save screening runs and evaluate them later against the latest snapshot.
- The user is new to the project and wants `alphasift quickstart` to demonstrate the minimal full-market → candidates → ranking loop in one step.

## Preconditions

- The default is `market="cn"`; `market="us"` is also available (yfinance snapshots and daily candlesticks, without CN-only sources such as hotspot/board_heat/tushare), but requires a strategy whose declared `market_scope` includes `us`, currently `us_momentum_quality`.
- Install the package from the repository root first: `pip install -e .`.
- For LLM ranking, configure `LITELLM_MODEL`, `LLM_CHANNELS`, `LITELLM_CONFIG`, or the legacy variables `LLM_API_KEY/LLM_MODEL/LLM_BASE_URL`
- You can directly reuse LiteLLM configuration fields from `daily_stock_analysis`, including `OPENAI_*`, `GEMINI_*`, `DEEPSEEK_API_KEY`, and `OLLAMA_API_BASE`
- Strategy YAML can override default rules through `scoring_profile`, `risk_profile`, `portfolio_profile`, and `scorecard_profile`.
- Strategy YAML can configure preferred/avoided events, announcement categories, and candidate-context source weights through `event_profile`.
- The LLM outputs candidate sector/theme labels. If candidates provide `industry/concepts/board_heat_score/board_heat_trend_score`, these anchor the LLM, theme-heat factors, and portfolio diversification layer; a history sidecar can supply persistence, cooling, and state fields. The default diversification layer maps these labels to risk buckets to reduce repeated exposure to the same crowded trade
- L3 enables the local `scorecard` post-scorer by default; `dsa` or `external_http` can also be added
- For DSA post-analysis, set `DSA_API_URL`; the default call is `POST /api/v1/analysis/analyze`.
- Strategies requiring daily candlesticks automatically enrich the Top N candidates after L1
- The daily candlestick setting `DAILY_SOURCE` supports `akshare` (default), `baostock`, or `auto`; `auto` automatically falls back to baostock as a free backup if akshare fails.

## Operations

### 1. List strategies

```bash
alphasift strategies
```

### 1.1 One-step demo (no API key)

```bash
alphasift quickstart
alphasift quickstart --strategy balanced_alpha --max-output 8
```

### 2. Run screening

```bash
alphasift screen dual_low --no-llm
alphasift screen volume_breakout --max-output 10
alphasift screen balanced_alpha --no-llm
alphasift screen capital_heat
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

### 3. Python API

```python
from alphasift import evaluate_saved_run, evaluate_saved_runs, list_strategies, screen

list_strategies()
screen("dual_low", market="cn", use_llm=False)
evaluate_saved_run("<run_id>")
evaluate_saved_runs(limit=20)
```

## Output

Returns `ScreenResult` JSON with the following core fields:
- `strategy`
- `market`
- `strategy_version`
- `snapshot_count`
- `after_filter_count`
- `picks`
- `llm_ranked`
- `llm_market_view`
- `llm_selection_logic`
- `llm_portfolio_risk`
- `llm_coverage`
- `post_analyzers`
- `daily_enriched`
- `risk_enabled`
- `portfolio_concentration_notes`
- `degradation`
- `snapshot_source`
- `source_errors`

Each `Pick` contains:
- `rank`
- `code`
- `name`
- `final_score`
- `screen_score`
- `ranking_reason`
- `risk_summary`
- `price`
- `change_pct`
- `amount`
- `total_mv`
- `turnover_rate`
- `volume_ratio`
- `pe_ratio`
- `pb_ratio`
- `industry`
- `concepts`
- `board_heat_score`
- `board_heat_latest_score`
- `board_heat_trend_score`
- `board_heat_persistence_score`
- `board_heat_cooling_score`
- `board_heat_observations`
- `board_heat_state`
- `board_heat_summary`
- `change_60d`
- `signal_score`
- `macd_status`
- `rsi_status`
- `breakout_20d_pct`
- `range_20d_pct`
- `volume_ratio_20d`
- `body_pct`
- `pullback_to_ma20_pct`
- `consolidation_days_20d`
- `factor_scores`
- `llm_confidence`
- `llm_sector`
- `llm_theme`
- `llm_tags`
- `llm_catalysts`
- `llm_risks`
- `llm_thesis`
- `llm_style_fit`
- `llm_watch_items`
- `llm_invalidators`
- `risk_score`
- `risk_level`
- `risk_penalty`
- `risk_flags`
- `portfolio_penalty`
- `portfolio_flags`
- `post_analysis_status`
- `post_analysis_score_deltas`
- `deep_analysis_status`
- `deep_analysis_summary`
- `deep_analysis_result`
- `deep_analysis_signal_score`
- `deep_analysis_sentiment_score`
- `deep_analysis_operation_advice`
- `deep_analysis_trend_prediction`
- `deep_analysis_risk_flags`

## Boundaries

- There is currently no standalone remote `get_result` service; manage run records locally with `--save-run`, `runs`, `evaluate`, and `evaluate-batch`.
- `audit` checks strategy profile coverage, known capability gaps, and next-step priorities.
- `--candidate-context-file` supports CSV/JSON/JSONL and aligns candidate-level news, announcements, fund flows, or research summaries by `code`, injecting only rows relevant to the current candidate pool. Optional fetching includes `source_count`, `source_confidence`, `source_weight_score`, `context_summary`, and announcement categories.
- Candidate-level context identifies broad event and negative-risk tags for cross-candidate LLM ranking.
- `industry-cache` caches industry/concept mappings and sector-heat fields and writes a history sidecar. Later mapping loads can supply rolling sector-heat trends, persistence, cooling, and state fields for LLM context and the `theme_heat` factor.
- The diversification layer prefers LLM sector/theme labels and can fall back to the candidate's `industry` field; if both are missing, it does not change the rule-based score.
- L3 post-analyzers run only on final candidates and do not participate in initial full-market screening. The local `scorecard` is enabled by default; DSA is one optional additional backend.
- T+N evaluation uses saved prices and the latest snapshot prices at evaluation time, rather than a full backtest using adjusted prices. It can deduct transaction costs and output retrospective breakout/pullback labels; `--with-price-path` also estimates maximum drawdown and maximum favorable excursion.
