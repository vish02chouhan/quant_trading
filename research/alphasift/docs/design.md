# Design principles

## 1. Code must enforce hard conditions

Code executes all deterministic conditions, including valuation, liquidity, price changes, and technical-indicator thresholds. Do not delegate verifiable logic to the LLM.

## 2. Default rules must be overridable

The system can provide default scoring curves and risk thresholds that work out of the box, but must not hardcode market-style assumptions. Strategy YAML can override default behavior through `scoring_profile`, `risk_profile`, `portfolio_profile`, and `scorecard_profile`.

## 3. Maintain strategy knowledge in one place

Define all stock-selection logic in the `screening:` section of `strategies/*.yaml`. Do not hardcode screening conditions in code.

## 4. Cross-candidate scoring is separate from single-stock diagnosis

`screen_score` is a cross-candidate score specifically for stock selection. It is not equivalent to `signal_score` or `sentiment_score` in single-stock analysis.

Reasons:
- Single-stock diagnostic scores answer "Is this stock worth analyzing?", rather than "How should candidates across the market be compared?"
- Reusing them directly would blur the responsibilities of L1/L2/L3.

## 5. Separate ranking from risk controls

Do not mix alpha ranking and risk constraints into one black-box total score. The risk overlay determines vetoes and penalties separately.

## 6. The LLM only makes qualitative judgments

The LLM handles: interpreting news, relative candidate ranking, candidate sector/theme labels, potential catalysts, risk summaries, and ranking explanations.
The LLM does not handle: initial full-market screening, numerical threshold checks, or exact technical-indicator calculations.

## 7. Portfolio constraints use LLM input without giving the LLM unrestricted control

The LLM can identify shared sectors/themes among candidates and report portfolio concentration risk. If candidates already carry `industry/concepts/board_heat_score/board_heat_trend_score/board_heat_persistence_score/board_heat_cooling_score`, these structured fields anchor the LLM's judgment. Code maps LLM labels or structured industries into sector risk buckets and converts them into an auditable `portfolio_penalty`. For example, banks, brokerages, and insurers all belong to the financial risk bucket. This uses the LLM's semantic classification ability without letting it directly determine hard sector quotas.

Industry/concept/sector-heat anchors primarily come from local CSV/JSON/JSONL mapping files. AkShare sector lookup can also be explicitly enabled. Full-market sector fetching is disabled by default to keep a single screening run from generating large numbers of network requests.

## 8. Start with A-share P0, then consider multiple markets

P0 targets A-shares only. Expand to Hong Kong/US stocks after confirming field mappings and data sources.

## 9. Graceful degradation

- Full-market snapshot unavailable → fail the task immediately (fail-fast).
- Essential fields missing → fail the task rather than degrade ambiguously.
- A single row fails daily candlestick enrichment required by the strategy → record degradation and reject that candidate in the daily hard filters; fail the task only if daily enrichment cannot run globally.
- Optional daily candlestick enrichment fails → fall back to snapshot scoring and record degradation.
- LLM ranking fails → fall back to ranking by screen_score.
- Optional L3 post-analyzer fails → fail the current task or record degradation, depending on whether the user explicitly requested the analyzer.

## 10. Decouple external analysis systems

alphasift handles L1 (hard filtering) and L2 (ranking). L3 is a post-analyzer framework that uses the local scorecard by default; external DSA or other HTTP scoring tools can also be added.

Here, DSA refers to the external project `daily_stock_analysis`:
- alphasift discovers and ranks candidates across the market.
- daily_stock_analysis performs in-depth single-stock analysis.
- The two are deployed separately and connected through `DSA_API_URL`.

To control costs, the default scorecard covers all final candidates; DSA is called only for the first N final shortlisted candidates. DSA is neither a core dependency for full-market screening nor the only L3 tool.

## 11. Stock selection must be evaluable

A screening run must save enough context for later evaluation:

- Strategy name and `strategy_version`.
- Data sources and degradation records.
- L1/L2/L3 scores and risk fields.
- Candidate prices at the time of saving.

Current T+N evaluation uses saved prices and the latest snapshot prices at evaluation time to calculate returns, win rates, missing quotes, transaction-cost deductions, and retrospective breakout/pullback pattern labels. Optional price paths also estimate maximum drawdown and maximum favorable excursion. This closes the engineering feedback loop; it is not a full backtest using adjusted prices.
