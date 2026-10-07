# Configuration reference

This document covers `.env`, LiteLLM, data sources, and L3 post-analyzer configuration. A minimal run does not require an LLM key; add `--no-llm` to the command.

## Minimal configuration

```bash
cp .env.example .env
```

Without LLM ranking:

```bash
alphasift screen dual_low --no-llm
```

For LLM ranking, enter a key from any supported provider and optionally specify the model:

```env
GEMINI_API_KEY=...
LITELLM_MODEL=gemini/gemini-2.5-flash
```

To use Tushare as a fallback data source, enter:

```env
TUSHARE_TOKEN=...
```

## Environment variables

| Variable | Required | Description | Default |
|------|------|------|--------|
| `LITELLM_MODEL` | Recommended | Main model, compatible with daily_stock_analysis, in `provider/model` format | `gemini/gemini-2.5-flash` |
| `LITELLM_FALLBACK_MODELS` | No | Comma-separated fallback models | - |
| `LLM_CHANNELS` | No | Multi-channel configuration, used with `LLM_{NAME}_*` | - |
| `LITELLM_CONFIG` | No | Path to an advanced LiteLLM Router YAML configuration file | - |
| `GEMINI_API_KEY` / `OPENAI_API_KEY` / `DEEPSEEK_API_KEY` | At least one for LLM ranking | Provider API key, compatible with daily_stock_analysis configuration | - |
| `OPENAI_BASE_URL` / `OLLAMA_API_BASE` | No | OpenAI-compatible API or Ollama endpoint | - |
| `LLM_API_KEY` | No | Legacy API key; takes precedence over provider keys | - |
| `LLM_MODEL` | No | Legacy model name; `LITELLM_MODEL` takes precedence | `gemini/gemini-2.5-flash` |
| `LLM_BASE_URL` | No | Legacy custom API endpoint | - |
| `LLM_CONTEXT` | No | Market, news, or theme context supplied to the LLM | - |
| `LLM_TEMPERATURE` | No | LLM ranking temperature; defaults favor deterministic output | `0.2` |
| `LLM_JSON_MODE` | No | Whether to request JSON response_format; falls back automatically if unsupported | `true` |
| `LLM_SILENT` | No | Whether to suppress LiteLLM call logs to avoid contaminating CLI JSON or summary output | `true` |
| `LLM_RANK_WEIGHT` | No | Weight of LLM ranking in the final score | `0.40` |
| `LLM_CANDIDATE_MULTIPLIER` | No | Number of candidates sent to the LLM as a multiple of max_output | `6` |
| `LLM_MAX_CANDIDATES` | No | Maximum candidates sent to the LLM per call | `30` |
| `LLM_MAX_RETRIES` | No | Retries when structured LLM output is invalid | `1` |
| `LLM_MIN_COVERAGE` | No | Minimum fraction of the candidate pool that LLM output must cover | `0.60` |
| `LLM_CONTEXT_MAX_CHARS` | No | Maximum length of combined context sent to the LLM | `4000` |
| `LLM_CANDIDATE_CONTEXT_ENABLED` | No | Whether to fetch news, announcements, and fund-flow clues for LLM Top K candidates by default | `false` |
| `LLM_CANDIDATE_CONTEXT_MAX_CANDIDATES` | No | Fetch candidate-level context for at most the first N stocks | `8` |
| `LLM_CANDIDATE_CONTEXT_PROVIDERS` | No | Comma-separated candidate-context sources: `news,fund_flow,announcement,quote` | `news,fund_flow,announcement,quote` |
| `LLM_CANDIDATE_CONTEXT_CACHE_ENABLED` | No | Whether to cache fetched candidate-level context | `true` |
| `LLM_CANDIDATE_CONTEXT_CACHE_TTL_HOURS` | No | Candidate-context cache lifetime in hours | `24` |
| `INDUSTRY_MAP_FILES` | No | Comma-separated local code->industry/concepts/board_heat CSV/JSON/JSONL mapping files | - |
| `INDUSTRY_PROVIDER` | No | Optional industry, concept, and sector-heat provider such as `akshare`; disabled by default | `none` |
| `INDUSTRY_PROVIDER_MAX_BOARDS` | No | Maximum sectors to look up in provider mode | `80` |
| `SNAPSHOT_SOURCE_PRIORITY` | No | Comma-separated data-source priority; if unset and a Tushare token is configured, `tushare` is preferred | Without token: `sina,efinance,akshare_em,em_datacenter` |
| `SNAPSHOT_FALLBACK_MAX_AGE_HOURS` | No | Maximum acceptable age of the last-good snapshot cache; empty/`none` means no limit | - |
| `ALPHASIFT_SOURCE_CALL_TIMEOUT_SEC` | No | Global waiting timeout for third-party data-source wrapper calls; `0`/`off` disables it | - |
| `ALPHASIFT_SNAPSHOT_CALL_TIMEOUT_SEC` | No | Call timeout for snapshot wrappers `efinance`/`akshare_em`/`tushare` | `60` |
| `ALPHASIFT_DAILY_CALL_TIMEOUT_SEC` | No | Call timeout for daily wrappers `akshare`/`baostock`/`tushare`/`yfinance` | `20` |
| `ALPHASIFT_EASTMONEY_MIN_INTERVAL_SEC` | No | Minimum serial interval between direct Eastmoney HTTP requests | `1.0` |
| `ALPHASIFT_EASTMONEY_JITTER_SEC` | No | Additional random jitter for each direct Eastmoney request | `0.3` |
| `TUSHARE_TOKEN` / `TUSHARE_API_TOKEN` | Required for `tushare` | Tushare Pro token for latest-trading-day daily and daily_basic fallback data | - |
| `TUSHARE_TRADE_DATE` | No | Fixed Tushare trading date in `YYYYMMDD` format for reproducible experiments | Automatically use the latest open trading day |
| `POST_ANALYZERS` | No | L3 post-analyzers; set to `none` to disable | `scorecard` |
| `POST_ANALYSIS_MAX_PICKS` | No | Maximum first N stocks processed by expensive L3 analyzers such as DSA/HTTP; the local scorecard processes all output by default | `3` |
| `POST_ANALYZER_URL` | Required for `external_http` | HTTP endpoint of an external scoring tool | - |
| `POST_ANALYZER_TIMEOUT_SEC` | No | External scoring-tool timeout in seconds | `120` |
| `DSA_API_URL` | Required for the `dsa` analyzer | DSA service URL or full analysis endpoint | - |
| `DSA_REPORT_TYPE` | No | DSA report type | `detailed` |
| `DSA_MAX_PICKS` | No | Perform in-depth analysis on at most the first N candidates | `3` |
| `DSA_TIMEOUT_SEC` | No | Timeout in seconds for one DSA request | `120` |
| `DSA_FORCE_REFRESH` | No | Whether to force DSA to ignore its cache | `false` |
| `DSA_NOTIFY` | No | Whether to allow DSA to send external notifications | `false` |
| `DAILY_ENRICH_ENABLED` | No | Whether to enrich L1 Top N candidates with daily candlestick features by default | `false` |
| `DAILY_ENRICH_MAX_CANDIDATES` | No | Maximum candidates processed by daily enrichment | `100` |
| `DAILY_LOOKBACK_DAYS` | No | Lookback days for daily features | `120` |
| `DAILY_SOURCE` | No | Daily candlestick source: `auto`, `tencent`, `sina`, `akshare`, `baostock`, or `tushare`; `auto` uses `tushare,tencent,sina,akshare,baostock` with a Tushare token, otherwise `tencent,sina,akshare,baostock` | `auto` |
| `DAILY_FETCH_RETRIES` | No | Retries after a candidate's daily candlestick fetch fails | `2` |
| `DAILY_FETCH_MAX_WORKERS` | No | Concurrent daily fetches; use `1` on unstable networks and `2`/`4` once stable | `1` |
| `RISK_ENABLED` | No | Whether to enable the independent risk layer | `true` |
| `RISK_MAX_PENALTY` | No | Maximum risk-layer penalty | `12` |
| `RISK_VETO_HIGH` | No | Whether to directly exclude high-risk candidates | `false` |
| `EVALUATION_COST_BPS` | No | Round-trip cost deducted from T+N evaluation returns, in bps | `0` |
| `EVALUATION_FOLLOW_THROUGH_PCT` | No | Minimum return percentage for a retrospective "breakout follow-through" label | `3` |
| `EVALUATION_FAILED_BREAKOUT_PCT` | No | Maximum return percentage for a retrospective "failed breakout" label | `-3` |
| `EVALUATION_PRICE_PATH_ENABLED` | No | Whether evaluation fetches daily price paths to calculate maximum drawdown and maximum favorable excursion | `false` |
| `EVALUATION_PRICE_PATH_LOOKBACK_DAYS` | No | Daily lookback days for price paths | `90` |
| `ALPHASIFT_DATA_DIR` | No | Directory for run records and evaluation results | `./data` |
| `STRATEGIES_DIR` | No | Strategy directory path | Auto-detected |

## LiteLLM configuration compatibility

AlphaSift follows the LiteLLM configuration conventions of `daily_stock_analysis`:

```env
LLM_CHANNELS=primary
LLM_PRIMARY_PROTOCOL=openai
LLM_PRIMARY_BASE_URL=https://api.deepseek.com/v1
LLM_PRIMARY_API_KEYS=sk-xxx,sk-yyy
LLM_PRIMARY_MODELS=deepseek-chat,deepseek-reasoner
LITELLM_MODEL=openai/deepseek-chat
LITELLM_FALLBACK_MODELS=openai/gpt-4o-mini,anthropic/claude-3-5-sonnet
```

A simpler single-provider configuration is also supported:

```env
GEMINI_API_KEY=...
LITELLM_MODEL=gemini/gemini-2.5-flash
```

If you already have a `daily_stock_analysis` `.env`, you can usually reuse fields such as `LITELLM_MODEL`, `LITELLM_FALLBACK_MODELS`, `LLM_CHANNELS`, `LLM_{NAME}_*`, `OPENAI_*`, `GEMINI_*`, `DEEPSEEK_*`, and `OLLAMA_API_BASE`.

The CLI also supports explicitly loading external `.env` files, with repeated arguments:

```bash
alphasift --env-file /path/to/daily_stock_analysis/.env \
  --env-file /path/to/daily_ai_assistant/.env \
  screen balanced_alpha
```

## Data-source configuration

Five A-share full-market snapshot sources are supported, with automatic fallback in priority order. Without a Tushare token, the default is:

```text
sina -> efinance -> akshare_em -> em_datacenter
```

If `TUSHARE_TOKEN` / `TUSHARE_API_TOKEN` is configured and `SNAPSHOT_SOURCE_PRIORITY` is not manually set, the default chain becomes:

```text
tushare -> sina -> efinance -> akshare_em -> em_datacenter
```

| Data source | API | Characteristics |
|--------|------|------|
| `sina` | vip.stock.finance.sina.com.cn | Direct full-market source, with PE/PB/turnover/market-cap fields |
| `efinance` | push2.eastmoney.com | Real-time push data; fastest during trading hours |
| `akshare_em` | 82.push2.eastmoney.com | Real-time push data; backup source |
| `em_datacenter` | data.eastmoney.com | Screener API; available outside trading hours |
| `tushare` | Tushare Pro `daily` + `daily_basic` | Latest-trading-day data; requires `TUSHARE_TOKEN`; not real-time |

When push2 APIs are unavailable on weekends or holidays, the system automatically falls back to `em_datacenter`. Direct Eastmoney APIs use a shared retry session, serial throttling, and random jitter to reduce connection instability and request bursts during repeated fallback. This follows `a-stock-data` recommendations for avoiding blocks with Eastmoney `em_get()`. AlphaSift adds caller-side timeouts to third-party wrapper calls such as efinance, AkShare, Baostock, Tushare, and yfinance; direct HTTP sources continue to use request-level timeouts. Repeated failures trigger a short-term source-health circuit breaker, temporarily skipping that source on subsequent runs. `doctor data-sources` outputs both raw `source_health` counters and a UI/agent-oriented `health_summary`, grouped into healthy/failing/disabled/never_seen. If all live daily sources fail, an expired but structurally valid history cache can be used, with `daily_stale/source_errors` marking degradation. If a source lacks fields required by the current strategy, such as PB, the system skips it and tries later sources. If all live sources fail, it can read `snapshot.last_good.json`, marking `fallback_used/stale/stale_age_hours/source_errors`. When `SNAPSHOT_FALLBACK_MAX_AGE_HOURS` is set, older caches are rejected to avoid repeatedly using outdated snapshots.

### Data-source capability matrix

| Capability | Default chain | Main fields |
|------|----------|----------|
| Daily candlestick enrichment | With token: `tushare,tencent,sina,akshare,baostock`; without token: `tencent,sina,akshare,baostock`, with unhealthy sources dynamically moved later based on source health | OHLCV, forward-adjusted prices where supported, technical indicators, 20-day volatility/ATR/drawdown, per-row `daily_source` provenance, `daily_quality_score`/flags (including the row-level `fetch_failed` flag), and source-health statistics. Screen degradation summarizes daily sources, quality flags, abnormal source health, and hard-filter rejection summaries. Low-quality, failed-fetch, or stale-cache rows contribute to final risk penalties |
| Full-market snapshot | With token: `tushare,sina,efinance,akshare_em,em_datacenter`; without token: `sina,efinance,akshare_em,em_datacenter` | Price, price change, trading value, market cap, PE/PB, turnover rate |
| Candidate-level context | `news,fund_flow,announcement,quote` | News, fund flows, announcements, Tencent quote valuation/turnover |
| Failure fallback | Source-health circuit breaker + daily history cache + snapshot last-good cache | stale/fallback/source_errors metadata |

## L3 post-analyzers

The local `scorecard` is enabled by default, providing stable, low-cost candidate-review scoring even without an external system.

Optional analyzers:

| Analyzer | Source | Purpose |
|---|---|---|
| `scorecard` | Local rule-based scoring | Enabled by default; lightweight score adjustments based on factors, LLM confidence, catalysts, and risks |
| `dsa` | External daily_stock_analysis | In-depth single-stock analysis of final candidates, extracting advice, trends, and risks |
| `external_http` | Custom HTTP tool | Connect other strategies, scorers, or research systems |

Example commands:

```bash
alphasift screen balanced_alpha
alphasift screen dual_low --post-analyzer dsa
alphasift screen capital_heat --post-analyzer external_http
alphasift screen balanced_alpha --no-post-analysis
```

`daily_stock_analysis` is not part of this repository. AlphaSift calls it through `DSA_API_URL` as an optional L3 backend, processing only final shortlisted candidates rather than participating in initial full-market screening.
