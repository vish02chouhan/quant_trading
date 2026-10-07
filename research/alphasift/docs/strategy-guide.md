# Strategy authoring guide

## File location

Place strategy files in `strategies/`; the filename is the strategy identifier (for example, `dual_low.yaml`).

## Minimal example

```yaml
name: my_strategy
display_name: My strategy
description: One sentence describing the strategy objective
version: "1.0"
category: value     # trend / value / income / quality / momentum / pattern / reversal

tags: [value, custom]
style:
  risk_profile: defensive
  holding_period: watchlist
  execution_style: mean_reversion
  market_regime: [risk_off, range_bound]
  capital_profile: medium_liquidity
  ui_badge: Value

screening:
  enabled: true
  market_scope: [cn]
  hard_filters:
    exclude_st: true
    amount_min: 50000000
  max_output: 5
```

## Complete schema

```yaml
name: string              # Unique identifier (English words with underscores)
display_name: string      # Display name
description: string       # Strategy description
version: string           # Strategy version; increment when semantics change
category: string          # trend / value / income / quality / momentum / pattern / reversal / framework
tags: [string]            # Optional tags for search and evaluation grouping
style:                    # Optional UI/agent strategy style; does not affect hard filtering
  risk_profile: string    # defensive / balanced / aggressive
  holding_period: string  # short_term / swing / watchlist
  execution_style: string # mean_reversion / momentum / breakout / multi_factor, etc.
  market_regime: [string] # risk_on / risk_off / trend / range_bound / rotation, etc.
  capital_profile: string # high_liquidity / medium_liquidity
  ui_badge: string        # Short label displayed in the UI

screening:
  enabled: bool            # Whether screening is enabled
  market_scope: [string]   # Applicable markets; currently only [cn]

  hard_filters:            # L1 hard conditions (all optional; omitted conditions do not filter)
    exclude_st: bool       # Exclude ST stocks
    price_min: float       # Minimum price
    price_max: float       # Maximum price
    amount_min: float      # Minimum trading value (yuan)
    market_cap_min: float  # Minimum total market cap
    market_cap_max: float  # Maximum total market cap
    pe_ttm_min: float      # Minimum PE(TTM)
    pe_ttm_max: float      # Maximum PE(TTM)
    pb_min: float          # Minimum PB
    pb_max: float          # Maximum PB
    volume_ratio_min: float    # Minimum volume ratio
    turnover_rate_min: float   # Minimum turnover rate
    change_pct_min: float      # Minimum price change
    change_pct_max: float      # Maximum price change
    change_60d_min: float      # Minimum 60-day gain
    change_60d_max: float      # Maximum 60-day gain
    require_ma_bullish: bool   # Require bullish moving-average alignment
    require_price_above_ma20: bool  # Require price above MA20
    signal_score_min: int      # Minimum signal score
    macd_status_whitelist: [string]  # Allowed MACD states
    rsi_status_whitelist: [string]   # Allowed RSI states

  tech_weight: float       # Technical-score weight, 0-1; default 0.35
  factor_weights:          # Optional multifactor weights; take precedence over tech_weight
    value: float           # Valuation
    liquidity: float       # Liquidity
    momentum: float        # Momentum
    reversal: float        # Reversal
    activity: float        # Activity
    stability: float       # Stability
    size: float            # Market-cap capacity
    theme_heat: float      # Theme/sector heat; optional soft factor

  scoring_profile:         # Optional overrides for default L1 factor-scoring curves
    momentum_chase_start_pct: float
    activity_ideal_volume_ratio: float
    activity_ideal_turnover_rate: float
    theme_heat_overheat_score: float
    theme_heat_trend_min_observations: float
    theme_heat_trend_slope: float
    theme_heat_cooling_penalty_slope: float
    theme_heat_persistence_min_score: float
    theme_heat_persistence_slope: float
    theme_heat_cooling_score_penalty_slope: float

  risk_profile:            # Optional overrides for risk-layer thresholds/penalties
    chase_change_pct: float
    abnormal_volume_ratio: float
    high_turnover_rate: float
    low_daily_quality_score: float
    fetch_failed_daily_points: float

  portfolio_profile:       # Optional overrides for LLM sector/theme risk buckets
    max_same_bucket: int
    concentration_penalty: float
    buckets:
      金融: [券商, 银行, 保险] # Financials: brokerages, banks, insurers; retain actual matching labels

  scorecard_profile:       # Optional overrides for default L3 scorecard adjustment rules
    value_quality_bonus: float
    volume_spike_ratio: float

  event_profile:           # Optional event preferences for LLM and candidate-context fetching
    preferred_event_tags: [string]
    avoided_event_tags: [string]
    preferred_announcement_categories: [string]
    avoided_announcement_categories: [string]
    source_weights:
      announcement: float
      news: float
      fund_flow: float

  ranking_hints: string    # Natural-language ranking guidance for the LLM
  max_output: int          # Final output count; default 5
```

`style` does not participate in hard filtering, but appears in `alphasift strategies --json` and strategy-matching commands. For example, `alphasift strategies --risk-profile defensive --market-regime risk_off --strict --json` returns candidate strategies with `score`, `matched`, and `missing` based on these fields, helping Web UIs, agents, or notification flows explain why a strategy was chosen.

## Strategy templates

When adding a strategy from scratch, start with a built-in template and adjust it for the target market environment, data-source coverage, and risk preferences:

```bash
alphasift strategies --templates --explain
alphasift strategies --template defensive_value_quality > strategies/my_defensive_value_quality.yaml
alphasift strategies --template momentum_breakout_daily --json
```

Current templates:

- `defensive_value_quality`: conservative value quality; snapshot-only; suitable for extending `quality_value` / `dual_low`.
- `momentum_breakout_daily`: daily volume breakout; requires `daily_k` and industry/theme context; run the data-source doctor before use.
- `oversold_reversal_snapshot`: oversold recovery; snapshot-only; a low-dependency starting point for reversal strategies when sources are unstable.

Templates are not stored under `strategies/*.yaml`, so they are not automatically loaded as enabled strategies. After writing one to `strategies/`, change `name`, `display_name`, and `version` first, then check field coverage with `alphasift doctor data-sources --strategy <name> --no-live --explain`.

## Strategy categories

| Category | Use case | Examples |
|---|---|---|
| `trend` | Trend confirmation and continuation | Low-volume pullback, volume breakout, bullish moving-average alignment |
| `value` | Valuation-driven screening | Dual low, high dividend yield, low PEG |
| `pattern` | Technical-pattern recognition | One bullish candle crossing three bearish candles, volume expansion at a bottom |
| `reversal` | Reversal-signal detection | Oversold rebound, bullish divergence at a bottom |

## Writing ranking_hints

`ranking_hints` is natural-language guidance sent to the LLM for relative candidate ranking.

Good practices:
- Explicitly list priority dimensions (1, 2, 3).
- Describe specific preferences, such as "clear volume contraction" rather than "good volume".
- Mention risk-exclusion conditions.
- Remind the LLM to identify shared sectors/themes to avoid a final list entirely exposed to one crowded trade.

Avoid:
- Letting the LLM invent selection criteria.
- Including exact numerical thresholds; these belong in hard_filters.
- Asking the LLM to provide target prices.

## Writing factor_weights

- Value strategies: increase `value` and `stability`, while retaining appropriate `liquidity`.
- Momentum strategies: increase `momentum`, `activity`, and `liquidity`.
- Capital/thematic strategies: add `theme_heat` to incorporate sector spread and warming/cooling trends into soft scoring.
- Reversal strategies: increase `reversal` and `stability` rather than simply chasing declines.
- General strategies: distribute weights across factors and let the LLM explain conflicts in L2.

## Writing profiles

Built-in default profiles provide only an out-of-the-box baseline. Whenever a threshold clearly represents a strategy preference, place it in YAML:

- Short-term heat strategies can lower `momentum_chase_start_pct` to penalize overextension earlier.
- Low-volatility value strategies can lower `chase_change_pct` and `volume_spike_ratio`.
- High-sensitivity thematic strategies can raise `high_turnover_rate` and `abnormal_volume_ratio`.
- Adjust sector concentration preferences with `portfolio_profile.buckets`, for example grouping banks, brokerages, and insurers into the financial risk bucket.
- Event-driven strategies can use `event_profile` to prefer buybacks, orders, and earnings improvements, and weight announcements more heavily than news.

Do not repeatedly tune profiles to extreme values as backtest-fitting parameters; they are better suited to expressing strategy style and risk preferences.

## Daily candlestick conditions

These fields require candidate-level daily enrichment:

- `change_60d_min` / `change_60d_max`
- `require_ma_bullish`
- `require_price_above_ma20`
- `signal_score_min`
- `macd_status_whitelist`
- `rsi_status_whitelist`
- `breakout_20d_pct_min` / `breakout_20d_pct_max`
- `range_20d_pct_max`
- `volume_ratio_20d_min` / `volume_ratio_20d_max`
- `body_pct_min` / `body_pct_max`
- `pullback_to_ma20_pct_min` / `pullback_to_ma20_pct_max`
- `consolidation_days_20d_min` / `consolidation_days_20d_max`

If a strategy configures these conditions, the pipeline first hard-filters snapshot fields, fetches daily candlesticks and calculates features for Top N candidates, then applies daily hard filters. This enables strategies such as `shrink_pullback` while avoiding individual historical-data fetches for the entire market.

## Versioning and evaluation

Update `version` when strategy semantics change, for example:

- Adjust hard-filter thresholds: `1.0` → `1.1`.
- Change factor-weight structure: `1.1` → `1.2`.
- Change the objective or use case: `1.x` → `2.0`.

Before submitting strategy changes, inspect parameter drift with comparison commands:

```bash
alphasift strategies --compare dual_low low_volatility_quality --explain
alphasift strategies --compare dual_low low_volatility_quality --json
```

The comparison payload lists differences in style, data dependencies, required fields, hard-filter parameters, factor weights, and profile keys. `summary.compatibility_notes` highlights changes in daily dependencies or data requirements.

`alphasift screen --save-run` saves `strategy_version`, candidates, scores, risks, and post-analysis results together. Later, `alphasift evaluate <run_id>` performs single-run retrospective T+N evaluation, while `alphasift evaluate-batch --limit 20 --explain` aggregates recent saved runs by strategy. Evaluation aggregates returns, transaction costs, sectors/themes, LLM catalysts/risks, post-analysis tags, risk tags, holding periods, and retrospective pattern labels. It outputs `failure_review` and `event_signal_review`, bringing together failed samples, shared event signals, event-signal win rates, common risk flags, failed breakouts, severe drawdowns, and next-step tuning suggestions.

## L3 post-analyzers

Strategy YAML controls only L1 filtering and basic scoring preferences, without binding a specific L3 tool. Choose tools at runtime through the CLI or environment variables:

```bash
alphasift screen balanced_alpha
alphasift screen dual_low --post-analyzer dsa
alphasift screen capital_heat --post-analyzer external_http
alphasift screen balanced_alpha --no-post-analysis
```

Available analyzers include:

- `scorecard`: lightweight local post-scoring, enabled by default and covering all output candidates.
- `dsa`: external daily_stock_analysis in-depth single-stock analysis, processing only the first N by default.
- `external_http`: custom HTTP scoring or research tool.
