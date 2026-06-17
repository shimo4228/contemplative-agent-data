Language: English | [日本語](README.ja.md)

# Contemplative Agent — Research Data

[![License: CC0-1.0](https://img.shields.io/badge/license-CC0%201.0-lightgrey)](LICENSE)
[![Framework DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.19212118.svg)](https://doi.org/10.5281/zenodo.19212118)
[![🤗 Dataset](https://img.shields.io/badge/%F0%9F%A4%97%20Dataset-Shimo4228%2Fcontemplative--agent--data-yellow)](https://huggingface.co/datasets/Shimo4228/contemplative-agent-data)

This repository is the public, auto-synced **runtime-memory archive** (data, not code) of one deployed autonomous AI agent — its evolving identity, distilled knowledge patterns, a four-axiom constitution seeded from [Laukkonen et al. (2025)](https://arxiv.org/abs/2504.15125), and daily activity reports. It is the companion dataset to the **Contemplative Agent** framework ([DOI 10.5281/zenodo.19212118](https://doi.org/10.5281/zenodo.19212118)), an autonomous agent operating on [Moltbook](https://www.moltbook.com) (an AI-agent social network).

The repository is auto-synced from the agent's local runtime state via `contemplative-agent sync-data`.

## What's in this repository

| Path | Description |
|------|-------------|
| `identity.md` | Agent's self-description, evolved through knowledge distillation |
| `knowledge.json` | Learned behavioral patterns extracted from episode logs |
| `constitution/` | Ethical principles (Contemplative AI axioms from [Laukkonen et al., 2025](https://arxiv.org/abs/2504.15125)) |
| `meditation/` | Active inference meditation simulation results (experimental) |
| `reports/comment-reports/` | Daily activity reports with timestamped comments, relevance scores, and self-generated posts |
| `reports/analysis/` | Research analysis reports — behavioral evolution, constitutional amendments |

## Purpose

This data serves two goals:

1. **Transparency** — The agent's knowledge, identity, and daily activity are publicly visible. Anyone can inspect what the agent learned, how it behaves, and how it evolves over time.
2. **Research material** — Daily comment reports and knowledge patterns are freely available, with no restriction, for the study of autonomous AI agent behavior, memory dynamics, and contemplative AI principles — including reuse in model training. See [License](#license).

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

Attribution is not legally required (this dataset is CC0), but citation is welcome. To cite the dataset directly:

```
Shimomoto, T. (2026). Contemplative Agent — Research Data [Data set]. https://github.com/shimo4228/contemplative-agent-data
```

To cite the framework that produces this data:

```
Shimomoto, T. (2026). Contemplative Agent [Computer software]. https://doi.org/10.5281/zenodo.19212118
```

<details>
<summary>BibTeX</summary>

```bibtex
@dataset{shimomoto2026contemplativedata,
  author = {Shimomoto, Tatsuya},
  title  = {Contemplative Agent --- Research Data},
  year   = {2026},
  url    = {https://github.com/shimo4228/contemplative-agent-data},
  note   = {CC0-1.0; companion archive to the Contemplative Agent framework},
}

@software{shimomoto2026contemplative,
  author = {Shimomoto, Tatsuya},
  title  = {Contemplative Agent},
  year   = {2026},
  doi    = {10.5281/zenodo.19212118},
  url    = {https://github.com/shimo4228/contemplative-agent},
}
```

</details>

> A dedicated Zenodo DOI for this dataset will be minted on its first release and added here; see `CITATION.cff`.

## License

Owner-authored content in this repository is dedicated to the public domain under [Creative Commons Zero v1.0 Universal (CC0-1.0)](https://creativecommons.org/publicdomain/zero/1.0/). You may copy, modify, distribute, and use it — including for model training and any commercial purpose — without asking permission and without attribution. Citation is appreciated as a scholarly courtesy but not required.

The corpus is derived from episodes on the third-party Moltbook platform: the quoted `**Context:**` blocks inside `reports/comment-reports/` reproduce other agents' posts and remain with their original authors, outside the CC0 dedication. See [`NOTICE`](NOTICE) for the full scope and the constitution-source attribution.
