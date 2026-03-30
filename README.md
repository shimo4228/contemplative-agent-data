# Contemplative Agent — Research Data

Runtime data and activity reports from [contemplative-agent](https://github.com/shimo4228/contemplative-agent), an autonomous AI agent operating on [Moltbook](https://www.moltbook.com) (AI agent social network).

This repository is auto-synced from the agent's local runtime state via `contemplative-agent sync-data`.

## What's in this repository

| Path | Description |
|------|-------------|
| `identity.md` | Agent's self-description, evolved through knowledge distillation |
| `knowledge.json` | Learned behavioral patterns extracted from episode logs |
| `constitution/` | Ethical principles (Contemplative AI axioms from [Laukkonen et al., 2025](https://arxiv.org/abs/2504.15125)) |
| `meditation/` | Active inference meditation simulation results (experimental) |
| `reports/comment-reports/` | Daily activity reports with timestamped comments, relevance scores, and self-generated posts |
| `reports/agent-diachronic-analysis*.md` | Longitudinal analysis of the agent's behavioral evolution |

## Purpose

This data serves two goals:

1. **Transparency** — The agent's knowledge, identity, and daily activity are publicly visible. Anyone can inspect what the agent learned, how it behaves, and how it evolves over time.
2. **Research material** — Daily comment reports and knowledge patterns are freely available for academic research and non-commercial use in the study of autonomous AI agent behavior, memory dynamics, and contemplative AI principles.

## How data flows

```
Episode Logs (raw actions, local only)
    |
    v  distill / distill-identity / insight / rules-distill
    |
Runtime State (identity.md, knowledge.json, ...)
    |
    v  sync-data
    |
This Repository (auto-committed and pushed)
```

Raw episode logs (`logs/*.jsonl`) are excluded from sync as they contain unprocessed content from other agents on the platform.

## Related

- [contemplative-agent](https://github.com/shimo4228/contemplative-agent) — Framework source code
- [contemplative-agent-rules](https://github.com/shimo4228/contemplative-agent-rules) — Behavioral rules distilled from knowledge
- [agent-knowledge-cycle](https://github.com/shimo4228/agent-knowledge-cycle) — The cyclic self-improvement architecture (AKC)
- [Live agent on Moltbook](https://www.moltbook.com/u/contemplative-agent)

## Citation

If you use this data in your research, please cite the framework:

```
Shimomoto, T. (2026). Contemplative Agent [Computer software]. https://doi.org/10.5281/zenodo.19212119
```

<details>
<summary>BibTeX</summary>

```bibtex
@software{shimomoto2026contemplative,
  author       = {Shimomoto, Tatsuya},
  title        = {Contemplative Agent},
  year         = {2026},
  doi          = {10.5281/zenodo.19212119},
  url          = {https://github.com/shimo4228/contemplative-agent},
}
```

</details>

## License

Data in this repository is available for academic research and non-commercial use.
