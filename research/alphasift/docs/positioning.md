# Positioning and strengths

## Positioning in one sentence

AlphaSift is a full-market candidate-discovery and cross-candidate ranking engine for AI agents: auditable rules first screen the full market, the LLM then performs semantic research, relative selection, risk attribution, and portfolio diversification, and pluggable post-analyzers provide a final review.

It is neither a single-stock diagnostic tool nor a simple wrapper around a traditional condition-based screener.

## Differences from traditional screeners

Traditional screeners are usually good at answering "Which stocks meet these conditions?", but weaker at semantic candidate comparison and portfolio tradeoffs. AlphaSift differs in the following ways:

- **Three separate layers of responsibility**: code enforces hard conditions, the LLM makes qualitative judgments, and L3 plugins perform in-depth analysis.
- **Cross-candidate comparison**: rather than assigning each stock an isolated score, it compares style, catalysts, risks, and crowding within the same candidate pool.
- **Structured LLM output**: requires `thesis`, `risk_flags`, `sector`, `theme`, `watch_items`, and `invalidators`, rather than only natural-language commentary.
- **Aligned candidate-level context**: `--candidate-context-file` injects news, announcements, fund flows, or research summaries aligned by stock code. Optional fetching includes source counts, source confidence, source weights, announcement categories, event tags, and compressed summaries so the LLM can compare candidates using supporting material.
- **Theme heat as a soft input**: industry/concept mappings can carry `board_heat_score`, rolling trends, persistence, cooling states, and heat summaries into factors and the LLM prompt, without acting as default hard filters.
- **Portfolio risk buckets**: the LLM identifies sectors/themes; code converts shared trades such as financials, consumption, AI computing, and new energy into auditable penalties.
- **Configurable rule profiles**: defaults are runnable, but strategy YAML can override scoring curves, risk thresholds, portfolio buckets, and scorecard rules.
- **Retrospective evaluation loop**: after saving runs, later snapshots can evaluate returns, win rates, and missing quotes, gradually testing strategies and LLM labels.

## Relationship with daily_stock_analysis

`daily_stock_analysis` is better suited to in-depth single-stock analysis, dashboards, and notifications for an existing stock pool/watchlist. AlphaSift sits upstream:

- AlphaSift discovers candidates across the market and ranks them against one another.
- daily_stock_analysis/DSA provides deeper single-stock analysis of final candidates.
- AlphaSift can use DSA as an L3 post-analyzer, but does not make it a main screening dependency.

The engineering reasons are cost and responsibility boundaries: full-market processing must be lightweight, fast, and able to fall back; in-depth single-stock analysis is suitable only for the small final shortlist.

## Relationship with daily_ai_assistant

`daily_ai_assistant` is closer to an LLM automation/notification assistant. AlphaSift can reuse its LLM configuration and multi-channel keys, but has different responsibilities:

- daily_ai_assistant focuses on task execution, message delivery, and automation.
- AlphaSift focuses on structured stock selection, candidate-pool management, ranking explanations, and the evaluation loop.

## Why the LLM is a core strength

AlphaSift does not use the LLM for deterministic screening or let it freely recommend stocks outside the pool. Its core value lies in areas traditional rules struggle to cover:

- Explain conflicting factors: attractive valuation but weak trend, strong capital flow but an overextended price, or a promising reversal with major fundamental risk.
- Distill the trade thesis: classify candidates into themes such as undervaluation recovery, capital-flow heat, consumer recovery, or AI computing.
- Identify shared risks: detect whether the final list is concentrated in financials, the real-estate supply chain, a single theme, or a single style.
- Identify events from context: turn buybacks, increased holdings, reduced holdings, regulatory inquiries, and earnings pressure in news/announcements/fund flows into candidate-level soft labels.
- Produce reviewable fields: `watch_items` and `invalidators` let later evaluations determine whether the original thesis was disproved.

## Current boundaries

- The current primary market is A-shares.
- This is not a full backtesting system using adjusted prices. T+N evaluation closes the engineering feedback loop; optional price paths add maximum drawdown and maximum favorable excursion, but do not handle portfolio rebalancing.
- The candidate pipeline supports `industry/concepts/board_heat_score` as anchors for the LLM, theme-heat factors, and portfolio risk buckets. These can be supplemented through local mapping files or optional AkShare sector lookup. `industry-cache` writes metadata and history JSONL; when mappings load, a same-name history sidecar supplies rolling trends, persistence, cooling, and state fields.
- Breakout, pullback, and other pattern strategies already enrich candidates with 20-day highs/ranges/volume/candle-body strength/pullback distance/platform duration. Evaluation can tag breakout follow-through/failure; with price paths enabled, maximum drawdown and maximum favorable excursion are available. Resistance density and intraday invalidation conditions still need to be added.
