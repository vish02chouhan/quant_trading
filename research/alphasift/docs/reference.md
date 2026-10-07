# Project reference

This document collects details that do not belong on the README front page: project structure, data-source boundaries, limitations, roadmap, and historical observed runs.

## Project structure

```text
alphasift/
├── SKILL.md                # Skill description for AI agents
├── strategies/             # Stock-screening strategy YAML
├── docs/
│   ├── configuration.md    # Configuration reference
│   ├── design.md           # Design principles
│   ├── positioning.md      # Project positioning
│   ├── scoring.md          # Scoring system
│   ├── strategy-guide.md   # Strategy authoring guide
│   └── usage.md            # Usage guide
└── alphasift/              # Python package
    ├── __init__.py
    ├── cli.py              # CLI entry point
    ├── config.py           # Environment configuration
    ├── context.py          # LLM context assembly
    ├── candidate_context.py # Candidate-level news/announcement/fund-flow context
    ├── daily.py            # Candidate-level daily candlestick enrichment
    ├── industry.py         # Industry/concept/sector-heat mapping
    ├── models.py           # Data models
    ├── snapshot.py         # Full-market snapshots, four sources + automatic fallback
    ├── filter.py           # L1 hard filters
    ├── scorer.py           # Score calculation
    ├── ranker.py           # L2 LLM ranking
    ├── risk.py             # Independent risk layer
    ├── post_analysis.py    # L3 pluggable post-analyzers
    ├── dsa.py              # Optional DSA integration
    ├── store.py            # Run-result persistence
    ├── evaluate.py         # Retrospective T+N evaluation and batch aggregation
    ├── pipeline.py         # Main orchestration
    └── strategy.py         # Strategy YAML loading
```

## Relationship with daily_stock_analysis

- `DSA` in the README, code, and environment variables refers to the external in-depth single-stock analysis service `daily_stock_analysis`.
- `alphasift` handles full-market candidate discovery, hard filtering, cross-candidate scoring, and LLM candidate ranking.
- `daily_stock_analysis` provides in-depth analysis of individual stocks, by default through `POST /api/v1/analysis/analyze`.
- The two are deployed separately and connected through `DSA_API_URL`. `daily_stock_analysis` is not part of this repository, but can serve as its L3 analysis backend.
- To control costs, `alphasift` calls DSA only for final shortlisted candidates. DSA's structured results affect `final_score`, risk judgments, and final positions at the last stage.
- The built-in `scorecard` is used by default; DSA or a custom `external_http` scorer can also be added.

## Data-source boundaries

Five A-share full-market snapshot sources are supported, with automatic fallback in priority order. If `SNAPSHOT_SOURCE_PRIORITY` is not explicitly set, the default without a Tushare token is `sina` -> `efinance` -> `akshare_em` -> `em_datacenter`; with a token it is `tushare` -> `sina` -> `efinance` -> `akshare_em` -> `em_datacenter`.

| Data source | API | Characteristics |
|--------|------|------|
| `sina` | vip.stock.finance.sina.com.cn | Direct full-market source, with PE/PB/turnover/market-cap fields |
| `efinance` | push2.eastmoney.com | Real-time push data; fastest during trading hours |
| `akshare_em` | 82.push2.eastmoney.com | Real-time push data; backup source |
| `em_datacenter` | data.eastmoney.com | Screener API; available outside trading hours |
| `tushare` | Tushare Pro `daily` + `daily_basic` | Latest-trading-day data; requires `TUSHARE_TOKEN`; not real-time |

When push2 APIs are unavailable on weekends or holidays, the system automatically falls back to `em_datacenter`. If a source times out, is unavailable, or lacks a required field such as PB, the system skips it and tries later sources. AlphaSift adds caller-side timeouts to wrapper sources such as efinance, AkShare, Baostock, Tushare, and yfinance. Source-health circuit breakers, daily history caches, and snapshot last-good caches continue to expose quality semantics such as `fallback_used/stale/stale_age_hours/source_errors`. If `SNAPSHOT_FALLBACK_MAX_AGE_HOURS` is set, older caches are not used.

## Tradeoffs informed by reference projects

- [`simonlin1212/a-stock-data`](https://github.com/simonlin1212/a-stock-data): explicitly prefers sources less prone to blocking, such as Tongdaxin/Tencent, uses Eastmoney only for unique data, and applies shared sessions, serial throttling, random jitter, and retries to direct Eastmoney calls. AlphaSift has adopted direct-HTTP-source priority, wrapper timeouts, a shared Eastmoney retry session, and `ALPHASIFT_EASTMONEY_*` throttling parameters.
- [`akfamily/akshare`](https://github.com/akfamily/akshare): broad coverage and simple calls, but official documentation emphasizes data risks and possible API changes. AlphaSift retains AkShare as a backup or optional provider rather than making it the only critical path.
- [`microsoft/qlib`](https://github.com/microsoft/qlib): emphasizes local data preparation, data-health checks, and reproducible research workflows. AlphaSift correspondingly adds `doctor data-sources`, source-health JSON, daily quality flags, and the saved-run/evaluate loop.
- [`ricequant/rqalpha`](https://github.com/ricequant/rqalpha) and [`zvtvz/zvt`](https://github.com/zvtvz/zvt): both decouple data and strategy layers, supporting extensible providers or local persistence before stock selection. AlphaSift keeps strategy YAML, source fallbacks, last-good caches, and upper-layer DSA/API integration separate instead of hardcoding a free source as a mandatory dependency.
- [`freqtrade/freqtrade`](https://github.com/freqtrade/freqtrade): makes strategy lists, backtesting, parameter optimization, and WebUI/status displays core workflows. AlphaSift is not a trading bot, but its strategy catalog needs machine-readable capability descriptions so the CLI, Web UI, DSA, or notification assistants can select strategies by data dependency and style.

## Known limitations

- Strategies requiring daily candlesticks enrich only Top N candidates after L1; this is neither a complete historical database nor a full-market backtesting system.
- The `dsa` post-analyzer depends on external `daily_stock_analysis` and currently calls synchronous REST requests one stock at a time, making it better suited to low-frequency in-depth analysis of the final list.
- L1/L2 main scoring still relies primarily on cross-sectional snapshots. Any L3 post-analyzer applies overlays and score adjustments only at the final stage, without participating in full-market initial screening.
- The `tushare` fallback source requires the user's own Pro token, API points, and permissions. It currently retrieves the latest trading day's close data rather than real-time order-book data.
- T+N evaluation uses prices at saving and the latest snapshot prices at evaluation, rather than a rigorous event backtest. It can deduct transaction costs, tag retrospective breakout/pullback patterns, and optionally fetch daily paths to estimate maximum drawdown/maximum favorable excursion, but does not yet handle dividends, suspensions, or rebalancing constraints.
- The repository keeps mirrored strategies in `strategies/` and `alphasift/strategies/` for development and installed usage. Built-in files must stay synchronized, while custom YAML can be added to `strategies/`.

## Improvement roadmap

Compared with similar intelligent investment-research projects, AlphaSift prioritizes:

- **Data reliability**: added `tushare` fallback, wrapper-call timeouts, source-health circuit breakers, `health_summary` aggregation, `freshness_summary` freshness/cache summaries, `--compare-snapshot-sources` multi-source field/code-intersection reconciliation, snapshot `quality_summary` anomaly reports, last-good/stale fallbacks, and `/data-source-history` aggregation of error/degradation/fallback rates from saved-run metadata. Next: visualize cache-hit trends.
- **Event-attribution loop**: LLM tags/catalysts/risks, post-analysis tags, and merged event signals are included in dimension statistics, `failure_review`, and `event_signal_review` from `evaluate-batch/evaluate-strategies`, identifying recurring signals in successful/failed samples and recommending prefer/avoid/watch actions.
- **Backtesting boundaries**: extend existing T+N evaluation with position constraints, rebalance periods, daily equity curves, and adjusted-price handling. Full quantitative research can integrate Qlib or Backtrader.
- **Agent artifacts**: added `alphasift overview --json/--explain`, `alphasift report <run_id>`, and the `alphasift serve` read-only local JSON API. These expose stable payloads for strategy groups, selection facets, cards, readiness, saved-run history summaries, source history, source health, recent runs, next actions, and single-run output for notification assistants, Web UIs, or MCP/HTTP services. Next: report templates and a more complete UI approval flow.
- **Strategy research**: added `alphasift strategies --json/--explain`, `strategies --compare`, `strategies --templates/--template`, `failure_review`/`event_signal_review` from `evaluate-batch/evaluate-strategies`, strategy styles, data dependencies, required snapshot/daily fields, single/all-strategy `doctor data-sources --strategy/--all-strategies` prechecks, `strategy_readiness_summary`, active-filter/factor-weight/profile metadata, and the defensive `low_volatility_quality` strategy. Event win-rate suggestions can produce strategy-level `screening.event_profile` YAML patches. Next: connect suggestions to UI approval or generate candidate strategy variants.

## Observed runs

### 2026-04-12 (Saturday, outside trading hours)

Test environment: Python 3.12; data from the previous trading day's close (2026-04-10).

- efinance / akshare real-time push APIs were unavailable outside trading hours; the run automatically fell back to `em_datacenter` (Eastmoney screener API).
- The current default chain supports Tushare and prefers it when a token is configured and `SNAPSHOT_SOURCE_PRIORITY` is not manually set. This recorded run had no Tushare token, so that source was not used.
- LLM ranking was disabled (`--no-llm`).

#### Dual Low (dual_low)

Full market: 5190 stocks -> 337 after hard filtering -> Top 5 output

| Rank | Code | Name | Score | Price | Change | PE | PB |
|------|------|------|------|------|--------|-----|-----|
| 1 | 002039 | 黔源电力 | 72.7 | 20.72 | -2.49% | 14.76 | 1.99 |
| 2 | 002444 | 巨星科技 | 71.0 | 30.82 | +0.29% | 14.59 | 1.95 |
| 3 | 002128 | 电投能源 | 70.9 | 31.60 | -2.41% | 14.00 | 1.90 |
| 4 | 002236 | 大华股份 | 70.8 | 17.43 | +1.04% | 14.86 | 1.50 |
| 5 | 600583 | 海油工程 | 68.9 | 7.02 | +4.15% | 14.89 | 1.17 |

#### Volume breakout (volume_breakout)

Full market: 5190 stocks -> 126 after hard filtering -> Top 5 output

| Rank | Code | Name | Score | Price | Change |
|------|------|------|------|------|--------|
| 1 | 002837 | 英维克 | 74.0 | 99.05 | +6.40% |
| 2 | 688183 | 生益电子 | 73.8 | 95.30 | +7.09% |
| 3 | 300803 | 指南针 | 73.3 | 101.68 | +3.07% |
| 4 | 002384 | 东山精密 | 73.0 | 143.55 | +8.83% |
| 5 | 300277 | 汽轮科技 | 73.0 | 19.74 | +5.73% |

#### Data-source fallback verification

| Data source | Status | Description |
|--------|------|------|
| efinance (push2.eastmoney.com) | Unavailable | Real-time push API returned an empty response outside trading hours |
| akshare_em (82.push2.eastmoney.com) | Unavailable | Same as above |
| em_datacenter (data.eastmoney.com) | Available | Screener API still returned the latest trading day's data on the weekend |
| tushare (Tushare Pro) | Not used | Currently supported; requires `TUSHARE_TOKEN` |

Fallback chain verified: `efinance` -> `akshare_em` -> `em_datacenter`, switching automatically to an available source.
