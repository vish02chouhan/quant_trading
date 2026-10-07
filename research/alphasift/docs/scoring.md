# Scoring system

## Composition of screen_score

`screen_score` is a cross-candidate score specifically for stock selection, used to rank candidates after L1 filtering.

| Component score | Description | Example factors |
|---|---|---|
| `factor_value_score` | Cross-candidate valuation score | PE and PB; lower positive values are better |
| `factor_liquidity_score` | Cross-candidate liquidity score | Trading value |
| `factor_momentum_score` | Cross-candidate momentum score | Daily price change, 60-day gain, signal_score, MACD |
| `factor_reversal_score` | Cross-candidate reversal score | Controlled declines, RSI, 60-day overheating/weakness |
| `factor_activity_score` | Cross-candidate capital-activity score | Volume ratio, turnover rate |
| `factor_stability_score` | Cross-candidate stability score | Extreme volatility, overheated turnover, negative PE, signal_score |
| `factor_size_score` | Cross-candidate capacity score | Total market cap |
| `factor_theme_heat_score` | Cross-candidate theme-heat score | `board_heat_score`, industry price change, industry rank, heat trend |

### Weights

The current implementation prefers `factor_weights` from strategy YAML. Available factors are:

| Factor | Meaning |
|---|---|
| `value` | Valuation attractiveness from low PE and low PB |
| `liquidity` | Tradability represented by trading value |
| `momentum` | Constructive positive price changes while avoiding extreme overextension |
| `reversal` | Recovery-watch value after a controlled decline |
| `activity` | Capital activity represented by volume ratio and turnover |
| `stability` | Penalties for extreme volatility, overheated turnover, and negative PE |
| `size` | Total market cap and capacity |
| `theme_heat` | Industry/concept/sector heat, with overheating penalties |

Example:

```
screen_score = Σ(factor_score × normalized_factor_weight)
```

If a strategy does not configure `factor_weights`, compatible weights are automatically mapped from `tech_weight`.

### Configurable scoring curves

Factors have default scoring curves, but strategies no longer need to share one set of hardcoded thresholds. Strategy YAML can override key parameters through `scoring_profile`, for example:

- `momentum_chase_start_pct`: price gain at which the momentum score starts penalizing overextension.
- `activity_ideal_volume_ratio`: preferred volume-ratio center for activity scoring.
- `activity_ideal_turnover_rate`: preferred turnover center for activity scoring.
- `reversal_ideal_change_pct`: preferred daily-decline center for reversal strategies.
- `stability_hot_change_pct`: price gain at which stability scoring starts penalizing overheating.
- `theme_heat_overheat_score`: threshold where theme-heat scoring starts penalizing overheating.
- `theme_heat_trend_min_observations`: minimum historical observations required to use sector-heat trends.
- `theme_heat_trend_slope` / `theme_heat_cooling_penalty_slope`: slopes for warming bonuses and cooling penalties.
- `theme_heat_persistence_min_score` / `theme_heat_persistence_slope`: threshold and slope for persistent-heat bonuses.
- `theme_heat_cooling_score_penalty_slope`: penalty slope for the latest cooling signal.

Default parameters represent a general baseline; strategy profiles express style differences. For example, short-term thematic trading can tolerate higher turnover, while conservative value strategies should penalize overheating earlier.

## Industry/concept/theme heat

`industry-cache` and local mapping files can provide:

- `industry` / `concepts`: structured industry and concept labels.
- `industry_rank` / `industry_change_pct`: cross-sectional sector rank and price change.
- `industry_heat_score` / `concept_heat_score` / `board_heat_score`: heat scores from 0 to 100.
- `board_heat_latest_score` / `board_heat_trend_score` / `board_heat_observations`: latest heat, rolling heat change, and observation count supplied by a history sidecar.
- `board_heat_persistence_score` / `board_heat_cooling_score` / `board_heat_state`: persistent heat, latest cooling magnitude, and state labels within a rolling window.
- `board_heat_summary`: readable heat summary, for example `银行:+1.20%:rank=3` (banks: +1.20%, rank 3).

These fields are not default hard-filter conditions. They enter the `theme_heat` factor, candidate-pool structure summary, and LLM prompt to distinguish sector-supported spread, sustained warming, persistent heat with marginal cooling, and isolated one-day spikes. Short-term strategies can give `theme_heat` greater weight; value/defensive strategies can assign no weight and leave it as soft information for the LLM.

Besides the main CSV/JSON, `alphasift industry-cache` writes:

- `*.meta.json`: refresh time, provider, number of sectors fetched, and error descriptions.
- `*.history.jsonl`: sector-heat snapshots from each refresh. Loading the main mapping automatically reads the same-name history sidecar to supply trend, persistence, cooling, and state fields.

## Daily candlestick enrichment

The default main pipeline depends only on the full-market snapshot. When a strategy declares the following fields, or `--daily-enrich` is explicitly enabled, the system adds daily features only to Top N candidates after L1 snapshot hard filtering:

- `change_60d`
- `ma_bullish`
- `price_above_ma20`
- `macd_status`
- `rsi_status`
- `signal_score`
- `breakout_20d_pct`
- `range_20d_pct`
- `volume_ratio_20d`
- `body_pct`
- `pullback_to_ma20_pct`

These features participate in subsequent hard filters and factor scoring, but daily enrichment is not a full-market historical-data scan.

Daily data is fetched per candidate, with `DAILY_FETCH_RETRIES` handling temporary network instability. A single failure is recorded in degradation and given default features without disrupting other candidates in the batch. If a strategy strictly requires daily data, failed candidates are naturally rejected by daily hard filters, avoiding selection when essential pattern conditions cannot be verified.

## L3 post-analyzers

L3 post-processes final candidates. The local `scorecard` is enabled by default, providing stable, low-cost candidate-review scoring even without an external system. Currently supported:

| Analyzer | Source | Purpose |
|---|---|---|
| `scorecard` | Local rule-based scoring | Enabled by default; lightweight score adjustments based on factors, LLM confidence, catalysts/risks |
| `dsa` | External daily_stock_analysis | In-depth single-stock analysis of final candidates, extracting advice, trends, and risks |
| `external_http` | Custom HTTP tool | Connect other strategies, scorers, or research systems |

The default `scorecard` covers all final output candidates; expensive backends such as `dsa` and `external_http` process only the first `POST_ANALYSIS_MAX_PICKS` candidates by default.

`scorecard` adjustment thresholds can also be overridden through `scorecard_profile`, including value-quality bonuses, volume-spike penalties, LLM confidence thresholds, and caps on catalyst/risk-tag adjustments.

DSA does not participate in L1 full-market screening. It is called only for final shortlisted candidates and used as an overlay at the last stage:
- `screen_score` still determines the main ranking before admission to the final list.
- DSA's returned `signal_score`, `sentiment_score`, `operation_advice`, trend judgment, and risk factors adjust the final `final_score`.
- DSA is therefore better suited to a low-frequency, expensive final-review layer than a high-frequency main-pipeline scorer.

In the current implementation, DSA does not participate in L1 full-market screening. It is called only for final shortlisted candidates and used as an overlay at the last stage:
- `screen_score` still determines the main ranking before admission to the final list.
- DSA's returned `signal_score`, `sentiment_score`, `operation_advice`, trend judgment, and risk factors adjust the final `final_score`.
- DSA is therefore better suited to a low-frequency, expensive final-review layer than a high-frequency main-pipeline scorer.

## LLM ranking (L2)

The LLM performs relative ranking only on Top K candidates. Inputs are:

1. Candidate screen_score and key indicators.
2. `ranking_hints` from strategy YAML.
3. Full-market snapshot summary, candidate-pool breadth, factor means, leading candidates by factor, and main-score distribution.
4. News/intelligence summaries, if available.
5. External candidate-level CSV/JSON/JSONL clues, if available, aligned by `code` through `--candidate-context-file`; only rows relevant to the current candidate pool are injected.
6. Optional Top K fetched clues: `--collect-candidate-context` or `LLM_CANDIDATE_CONTEXT_ENABLED=true` fetches news, announcements, and fund-flow summaries with `source_count`, `source_confidence`, `source_weight_score`, `context_summary`, announcement categories, event tags, and negative-risk tags.

LLM outputs:

1. Global `market_view`: whether the current candidate pool and market context suit the strategy.
2. Global `selection_logic`: the main judgment dimensions for this ranking.
3. Global `portfolio_risk`: shared or concentration risks in the final list.
4. Reranked positions.
5. Each candidate's `thesis`, ranking explanation, risk summary, and potential catalysts.
6. Candidate `sector` / `theme` for portfolio concentration controls and later review attribution.
7. Candidate tags, risk tags, strategy-style fit explanations, and confidence.
8. `watch_items` and `invalidators`: follow-up observations and conditions that would disprove the candidate thesis.

The LLM can only rerank within the pool. It cannot recommend stocks outside it or replace hard-filter conditions.
LLM output undergoes JSON parsing, stock-code coverage validation, and duplicate/unknown-code checks. If thresholds are not met, it retries; continued failure falls back to `screen_score`. The call layer supports LiteLLM JSON mode, fallback models, multi-channel key/base_url resolution, and advanced Router YAML.

## Retrospective evaluation and pattern labels

`alphasift evaluate` and `evaluate-batch` calculate T+N returns from saved prices and the latest snapshot prices; `EVALUATION_COST_BPS` deducts round-trip costs. Candidates carrying daily pattern fields also receive:

- `breakout_follow_through`: a breakout candidate reaches `EVALUATION_FOLLOW_THROUGH_PCT`.
- `failed_breakout`: a breakout candidate falls below `EVALUATION_FAILED_BREAKOUT_PCT`.
- `breakout_unconfirmed`: between those thresholds.
- `pullback_rebound` / `pullback_failed`: retrospective performance of MA20 pullback candidates.

Batch evaluation aggregates by `by_shape_status` and `by_shape_tag` to reveal patterns that repeatedly fail. `evaluate-batch` also outputs `portfolio_summary` and `portfolio_by_strategy`, treating each run as an equal-weight portfolio to calculate portfolio returns, win rates, and, when price paths are enabled, portfolio-level average maximum drawdown/maximum favorable excursion.

JSON payloads from `evaluate-batch` and `evaluate-strategies` also include `failure_review` for strategy research:

- `summary`: failed sample count, negative-return count, missing-quote count, failed-breakout count, severe-drawdown count, and worst return.
- `failure_samples`: samples ranked by severity, including run, strategy, code, return, LLM tags/catalysts/risks, post-analysis tags, pattern state, risk/portfolio flags, `event_signals`, and failure reasons.
- `dimensions`: failed samples aggregated by strategy, industry/theme, LLM catalysts/risks, post-analysis tags, merged event signals, risk flags, portfolio flags, pattern states, and failure reasons.
- `recommendations`: next steps for parameter tuning and data checks.

Use `--failure-samples N` to control how many samples remain in explain/JSON output; `0` keeps only aggregates and recommendations.

The same batch-evaluation payload also outputs `event_signal_review`, unifying `llm_tags`, `llm_catalysts`, `llm_risks`, and `post_analysis_tags` into four event-signal types: `tag:`, `catalyst:`, `risk:`, and `post:`. Each signal includes sample count, win rate, mean/median/best/worst return, failure rate, and a `prefer` / `avoid` / `watch` recommendation, helping feed retrospective results back into strategy `preferred_event_tags`, `avoided_event_tags`, or the risk profile.

`event_signal_review.strategy_patch_suggestions` further aggregates this evidence by strategy, generating reviewable `screening.event_profile` YAML snippets and `append_unique` field-change suggestions. It only provides suggestions and does not automatically rewrite strategy files; review them in a UI/PR before adding stable signals to specific strategies.

When `--with-price-path` or `EVALUATION_PRICE_PATH_ENABLED=true` is enabled, evaluation also fetches candidate daily price paths and calculates:

- `path_end_return_pct`: return on the path's last trading day relative to the saved price.
- `max_drawdown_pct`: maximum downward excursion along the path relative to the saved price.
- `max_runup_pct`: maximum upward excursion along the path relative to the saved price.
- `path_status`: whether the path is available.

This is still not a full backtest, but addresses the gap of seeing only the final day's return without knowing interim drawdowns.

### LLM configuration

To make `daily_stock_analysis` configuration reusable, AlphaSift supports the same LiteLLM environment variables:

- `LITELLM_MODEL`
- `LITELLM_FALLBACK_MODELS`
- `LLM_CHANNELS` + `LLM_{NAME}_PROTOCOL/BASE_URL/API_KEY/API_KEYS/MODELS/ENABLED`
- `LITELLM_CONFIG`
- `OPENAI_API_KEY` / `OPENAI_BASE_URL`
- `GEMINI_API_KEY` / `GEMINI_API_KEYS`
- `DEEPSEEK_API_KEY`
- `OLLAMA_API_BASE`

Legacy variables `LLM_API_KEY`, `LLM_MODEL`, and `LLM_BASE_URL` remain supported.

## Risk overlay

The risk overlay is independent of strategy factors and the LLM. The current implementation applies penalties based on the following fields; `RISK_VETO_HIGH=true` can directly exclude high-risk candidates:

| Check | Behavior |
|---|---|
| Excessive daily gain or decline | penalty / optional veto |
| Abnormal volume ratio, high turnover | penalty / optional veto |
| Negative PE, high PB | penalty |
| Weak daily signal, bearish MACD, overheated RSI | penalty |
| Low daily data quality, failed fetch, expired cache, or source fallback | penalty / optional veto |
| LLM risk tags, low confidence | penalty |
| Risk tags from DSA or other post-analyzers | Enter candidate risk fields; DSA also affects post-analysis scores |

Final output includes `risk_score`, `risk_level`, `risk_penalty`, and `risk_flags` for subsequent agent or human review.

Strategy YAML can override risk thresholds and penalty points through `risk_profile`, for example `chase_change_pct`, `abnormal_volume_ratio`, `high_turnover_rate`, `low_llm_confidence`, `low_daily_quality_score`, and `fetch_failed_daily_points`. This avoids hardcoding market-style and data-quality assumptions such as "8% is overextended", "a volume ratio of 6 is abnormal", or "how many points a failed fetch costs".

## Portfolio diversification overlay

The LLM outputs standardized `llm_sector` and `llm_theme` for candidates. If a candidate snapshot or external data supplies `industry/concepts`, those fields enter LLM context; when LLM industry labels are missing, diversification uses `industry` as a fallback anchor. By default, AlphaSift maps these labels into portfolio risk buckets and applies a diversification overlay before trimming the final Top N. For example, banks, brokerages, and insurers all enter the financial risk bucket.

- Once a shared LLM sector/risk bucket exceeds `PORTFOLIO_MAX_SAME_LLM_SECTOR`, subsequent candidates receive a `PORTFOLIO_CONCENTRATION_PENALTY` deduction.
- The layer applies when the LLM returns sector labels or candidates provide structured `industry`; if both are absent, rankings are unchanged.
- Penalty records are written to `portfolio_penalty`, `portfolio_flags`, and `portfolio_concentration_notes`.

This is a mild constraint against one crowded trade filling the final list, rather than a hard sector quota; repeated-sector candidates can remain if their advantage is strong enough.

Default risk buckets can be overridden or extended through strategy YAML `portfolio_profile.buckets`. For example, a cyclical strategy can group steel, coal, and nonferrous metals into one cyclical bucket; an AI strategy can group computing infrastructure, optical modules, and servers into one trade bucket.
