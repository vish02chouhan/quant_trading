# English documentation

This local English edition translates the repository's Chinese documentation. Start with the [project README](../README.md), then use the guides below.

## Reading guide

- [Positioning and strengths](positioning.md): what AlphaSift is intended to do.
- [Design principles](design.md): responsibilities of screening, ranking, and analysis layers.
- [Usage guide](usage.md): installation, commands, Python calls, and saved-run evaluation.
- [Configuration reference](configuration.md): environment variables, providers, and post-analyzers.
- [Strategy files](../strategies/README.md): built-in strategies and examples.
- [Strategy authoring guide](strategy-guide.md): YAML fields, templates, and strategy versioning.
- [Scoring system](scoring.md): factors, LLM ranking, risk overlays, and evaluation metrics.
- [Result schema](result-schema.md): structured output contracts (already in English upstream).
- [Comparison and gaps](comparison.md): comparisons and improvement priorities.
- [Project reference](reference.md): layout, data-source boundaries, limitations, and recorded runs.
- [Changelog](CHANGELOG.md): upstream change history.
- [Root agent skill](../SKILL.md) and [GitHub agent skill](../.github/skills/alphasift/SKILL.md): translated usage instructions for agents.

## Engineering plans

- [Topic-first hotspots](plans/2026-06-05-topic-first-hotspots.md).
- [Hotspot stability P15](plans/2026-06-05-hotspot-stability-p15.md) (already in English upstream).
- [Performance optimization roadmap](plans/2026-06-05-performance-optimization-roadmap.md) (already in English upstream).

## Translation details

The source snapshot is upstream commit `7639195e6176f4af37086c282834a7c09dc3e4a3`. Translation was prepared on the source checkout's `english-docs` branch and is included here as a snapshot. Original files remain available in upstream Git history; the [Chinese README](../README.zh-CN.md) is retained unchanged. See [source and translation provenance](../UPSTREAM.md) for exact versions.

Only Markdown documentation changes. Code, strategy YAML files, configuration files, identifiers, numerical thresholds, and sample results are preserved. Natural-language text in documentation examples is translated. Chinese stock names, provider field aliases, and actual sector-matching labels remain where needed, with English explanations for the latter two.

Upstream claims, dates, historical observations, and stated limitations are translated as written. Translation does not update or independently verify them; older documents can therefore differ from current code or newer documentation.
