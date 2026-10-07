# Comparison and gaps

## Projects compared

| Project | Main positioning | Strengths | Relationship to AlphaSift |
|---|---|---|---|
| AlphaSift | Full-market candidate discovery and cross-candidate LLM ranking | A-share full-market hard filters, structured LLM reranking, portfolio risk buckets, pluggable L3, lightweight T+N evaluation | This project |
| daily_stock_analysis | Watchlist/single-stock in-depth analysis and notifications | Single-stock analysis, dashboards, notifications, multi-market watchlists | After AlphaSift screens candidates upstream, DSA can provide L3 in-depth analysis |
| daily_ai_assistant | LLM assistance, automation, and notifications | Multi-channel LLM configuration, task automation, message delivery | AlphaSift can reuse its LLM configuration, but does not act as a notification assistant |
| [OpenBB](https://docs.openbb.co/odp) | Financial data platform and research workflows | Multiple data sources and consumption layers such as Python/CLI/REST/MCP/Workspace | AlphaSift is not a general financial data platform; data-source breadth is a gap |
| [Microsoft Qlib](https://www.microsoft.com/en-us/research/publication/qlib-an-ai-oriented-quantitative-investment-platform/) | AI quantitative research platform | Data, models, backtesting, quantitative research workflows | AlphaSift does not replace Qlib; evaluation/backtesting could later move toward Qlib |
| [FinGPT](https://ai4finance.org/research/fingpt-open-source-finllm.html) | Financial LLM framework and model ecosystem | Financial LLM data, training, adaptation, deployment | AlphaSift uses general LLM interfaces rather than training financial foundation models |
| [Backtrader](https://www.backtrader.com/) | Python backtesting and trading framework | Strategy backtesting, indicators, analyzers, trading simulation | AlphaSift currently provides only lightweight T+N evaluation; full backtesting is a gap |

## AlphaSift's strengths

1. **Clear positioning**: full-market candidate discovery rather than single-stock analysis or notification assistance.
2. **Restrained LLM use**: the LLM only ranks and performs semantic attribution within the candidate pool; it does not participate in hard filtering or recommend stocks outside the pool.
3. **Structured output**: every candidate has factor scores, an LLM thesis, risks, catalysts, sectors/themes, watch items, and invalidators.
4. **Auditable portfolio risk**: the LLM supplies sectors/themes; code maps risk buckets and records `portfolio_penalty`.
5. **Theme heat informs qualitative judgment**: industry/concept mappings can carry `board_heat_score`, rolling trends, persistence, cooling states, and heat summaries into factors, the LLM prompt, and subsequent attribution.
6. **Runnable defaults with overridable rules**: strategy YAML can override scoring curves, risk thresholds, portfolio buckets, and the scorecard.
7. **Downstream integration**: DSA and external HTTP scorers are L3 post-analyzers that do not intrude into the main screening path.

## Current gaps

| Gap | Impact | Direction for improvement |
|---|---|---|
| Limited industry/concept data sources | Local mapping files, optional AkShare sector lookup, `industry-cache` refreshes, metadata/history sidecars, sector-heat scores, theme-heat summaries, rolling trends, persistence, cooling signals, and abnormal-heat-value filtering are supported, but source breadth remains limited | Add multi-source mappings, normalize sector hierarchies, and provide more complete data-quality reports |
| Limited news/announcement/fund-flow context | Candidate-level context files, optional Top K fetching, caching, basic deduplication, source confidence, source-weight scores, compressed summaries, announcement categories, event tags, negative-risk recognition, and strategy-level event preferences are supported, but event effectiveness has not entered retrospective attribution | Include event-type preferences in T+N attribution statistics; add finer announcement categories and source-quality evaluation |
| Incomplete backtesting | Saved-run batch T+N aggregation, strategy/industry/theme/risk-tag/holding-period dimensions, equal-weight portfolio summaries, transaction-cost deductions, and optional daily price-path maximum drawdown/maximum favorable excursion are supported, but cannot replace a backtest with adjusted prices and position constraints | Add position constraints and daily portfolio equity curves; later integrate Qlib/Backtrader |
| Coarse pattern validation | 20-day breakout strength, range amplitude, volume ratios, candle-body strength, MA20 pullback distance, platform duration, T+N retrospective pattern labels, and optional price-path drawdown/favorable excursion are supported, but resistance density and intraday invalidation conditions are absent | Add prior-high resistance density, intraday invalidation conditions, and portfolio position paths |
| Less data-source breadth than OpenBB | sina, efinance, akshare_em, em_datacenter, Tushare fallbacks, and multi-source daily candlestick degradation exist, but the system is not yet suited to general multi-asset, multi-market research | First reconcile A-share fields across sources and add quality reports, then expand to Hong Kong/US stocks |
| Missing UI/notification layer | Unsuitable as a daily notification product on its own | Integrate with daily_ai_assistant or other notification systems through decoupled interfaces |

## Priorities for closing gaps

1. **Industry/concept mapping**
   Candidates, LLM context, and portfolio diversification already support `industry/concepts/board_heat_score`. `industry-cache` refreshes local caches and writes metadata/history; loading mappings supplies rolling trends, persistence, cooling, and state fields and skips abnormal heat values. Next, normalize sector hierarchies and add more complete data-quality reports.

2. **Candidate-level news/announcement/fund-flow summaries**
   Context can currently be injected using `--candidate-context-file`, or fetched and cached for Top K using `--collect-candidate-context`. Fetched results include source counts, source confidence, source-weight scores, compressed summaries, announcement categories, event tags, and negative-risk recognition. Strategy YAML can configure preferences and source weights through `event_profile`. Next, include event-type preferences in T+N attribution statistics.

3. **Daily candlestick enrichment for pattern strategies**
   `volume_breakout` and `shrink_pullback` already use 20-day breakout/range/volume/candle-body/pullback/platform-duration fields. `evaluate` tags breakout follow-through/failure; enabling price paths produces maximum drawdown/maximum favorable excursion. Next, add resistance density and intraday invalidation conditions.

4. **Batch evaluation reports**
   Aggregation by strategy, industry, theme, tag, risk flag, portfolio flag, holding period, and equal-weight portfolio is available, with transaction-cost deductions. Price paths estimate maximum drawdown and maximum favorable excursion. Next, add position constraints and daily portfolio equity curves.

5. **External framework integration**
   Qlib/Backtrader are suitable full-backtest backends, but should not enter the main screening pipeline.

## Self-check commands

```bash
alphasift audit
alphasift audit --json
```

The self-check reports:

- Strategy counts and categories.
- Coverage of the four profile types.
- Strategy-level configuration gaps.
- Project-level gaps.
- Next-step priorities.
