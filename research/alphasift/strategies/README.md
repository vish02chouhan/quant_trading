# Strategy files

This directory contains stock-screening strategy YAML files.

## File format

Each `.yaml` file defines a screening strategy. Its `style:` section describes style attributes used by a UI/agent to choose strategies, and its `screening:` section defines screening rules.

See the [strategy authoring guide](../docs/strategy-guide.md).

Built-in strategies configure not only `hard_filters` and `factor_weights`, but also style-specific profiles:

- `scoring_profile`: factor-scoring curves, such as overextension penalties and ideal volume ratios/turnover.
- `risk_profile`: risk thresholds, such as abnormal volume ratios, high turnover, and low confidence.
- `portfolio_profile`: LLM sector/theme risk buckets.
- `scorecard_profile`: default L3 scorecard rules for adding and subtracting points.

Built-in strategies follow common multifactor ranking / sector-neutral screening practices: a style is not ranked using only one static indicator. Value strategies also use momentum, activity, or reversal confirmation; trend/short-term strategies add stability and theme-heat constraints. The portfolio layer uses `portfolio_profile` to limit repeated representation of the same risk bucket.

`capital_heat`, `balanced_alpha`, `momentum_quality`, and `volume_breakout` include an optional `theme_heat` factor. When local mappings or `industry-cache` provide `board_heat_score`, `board_heat_trend_score`, `board_heat_persistence_score`, and `board_heat_cooling_score`, sector/theme heat, persistence, and warming/cooling trends feed soft scoring and LLM context.

## Available strategies

| File | Name | Category | Description |
|------|------|------|------|
| `shrink_pullback.yaml` | Low-volume pullback | trend | Candidate-level daily candlestick enrichment after L1 identifies bullish moving-average alignment and pullback structures |
| `dual_low.yaml` | Dual Low | value | Starts with low PE + low PB and adds activity/momentum/reversal confirmation to reduce repeated dominance by static low-valuation stocks |
| `blue_chip_income.yaml` | Blue-chip income quality | income | Snapshot-only defensive candidates among liquid large-cap blue chips and dividend assets |
| `volume_breakout.yaml` | Volume breakout | trend | Breakouts above key resistance on increased volume, combined with theme heat and overextension penalties |
| `quality_value.yaml` | Quality value | value | Reasonable valuation, sufficient liquidity, and moderate volatility, with mild dynamic confirmation |
| `low_volatility_quality.yaml` | Low-volatility quality | quality | Defensive candidates with low volatility, shallow drawdowns, moderate valuation, and reliable data quality |
| `capital_heat.yaml` | Capital heat | momentum | Active capital and aligned price/volume without extreme overheating, avoiding overfitting to high-turnover spikes |
| `oversold_reversal.yaml` | Oversold reversal | reversal | Recovery candidates with controlled declines and intact liquidity, plus moderate activity confirmation |
| `balanced_alpha.yaml` | Balanced multifactor | framework | Combines valuation, capital, momentum, stability, reversal, and theme heat |
| `momentum_quality.yaml` | Momentum quality | framework | Medium-term candidate discovery combining trend confirmation, quality constraints, theme heat, and portfolio diversification |

## Example strategies (optional, excluded from built-ins)

The `examples/` subdirectory contains optional example strategies. They are not loaded automatically and do not change the built-in list. To use one, copy it into this directory; the repository's local custom-strategy mechanism will detect it automatically:

| File | Name | Category | Description |
|------|------|------|------|
| `examples/dual_low_us.yaml` | Dual Low (US) | value | Low PE + low PB value screen for US stocks (`market_scope: [us]`; requires `yfinance` and `--market us`) |

```bash
cp strategies/examples/dual_low_us.yaml strategies/
alphasift screen dual_low_us --market us --no-llm
```

## Running and evaluating

Strategies that require daily candlesticks automatically apply lightweight enrichment to Top N candidates after L1, including MA, MACD/RSI, 20-day breakout strength, range amplitude, 20-day volume ratio, candle-body strength, MA20 pullback distance, and platform duration:

```bash
alphasift screen shrink_pullback --no-llm
```

Update the YAML `version` when strategy semantics change. Saved runs support retrospective T+N evaluation, which labels patterns such as breakout follow-through, failed breakout, and MA20 pullback recovery:

```bash
alphasift screen balanced_alpha --no-llm --save-run
alphasift evaluate <run_id> --explain
```
