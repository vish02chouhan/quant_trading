# Changelog

## Unreleased

- Support candidate providers injected by DSA through `context["dsa"]`. AlphaSift adds DSA quote, fundamental, and news context after initial L1 screening and before LLM reranking.
- `dsa_adapter.screen()` now forwards DSA context and preserves `dsa_context`, `dsa_news`, and `dsa_analysis_summary` in candidate results.
- The LLM ranking prompt reads candidates' DSA provider context, allowing ranking to use DSA's existing data capabilities.

## 2026-04-12

- Clarify that `DSA` refers to the external project `daily_stock_analysis`, and document responsibility boundaries and call relationships.
- Update the README, Skill documentation, and design notes to explain that DSA is called only for final shortlisted candidates.
- Correct outdated documentation: remove incorrect claims that `shrink_pullback` can run directly and that L3 deep_analysis is unimplemented.
- Document current DSA overlay behavior: structured results affect `final_score`, risk judgments, and final rankings at the last stage.
