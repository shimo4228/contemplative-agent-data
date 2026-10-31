# `reports/analysis/` — file formats

This file is hand-kept. The rest of this directory mirrors the framework's runtime
and is rewritten by the daily sync. Dates are those of the files present on
2026-10-31; "active" means the weekly chain still writes the format.

| Format | Status | Files present (first to last) |
|---|---|---|
| `weekly-DATE.md` | active | 2026-03-30 to 2026-10-24 |
| `weekly-DATE-findings.md` | active | 2026-05-17 to 2026-10-24 |
| `weekly-DATE-archive-candidates.txt` | active | 2026-09-11 to 2026-10-24 |
| `observation-ledger.jsonl` | active (append-only) | — |
| `.anomaly-sweep-state.tsv` (+ `.corpus.tsv`), `.api-drift-state.tsv`, `.approval-join-state.json` | active (pipeline state) | — |
| `weekly-DATE.ja.md`, `weekly-DATE-findings.ja.md` | retired: Japanese translations ended with the one-session report (ADR-0098, ADR-0099) | 2026-07-05 to 2026-08-21 |
| `weekly-DATE-packet.md` | retired: decision packets | 2026-07-24 to 2026-08-21 |
| `weekly-DATE-insight-review.md` | retired | 2026-07-24 to 2026-08-21 |
| `weekly-DATE-improvement.md` | retired | 2026-07-24 only |
| `agent-diachronic-analysis(-ja).md` | not updated since 2026-03-28: the initial 14-day analysis | — |
| `constitution-amendment-report.md` | not updated: the first amendment only | — |
| `patches/` | removed from the mirror (decision D-P3-05-4 = A; framework code under MIT); in git history | 2026-07-31 to 2026-08-21 |
| `../../pipeline/ledger-watch/ledger-watch-*.json` | retired (ADR-0095), removed from the tree; in git history | 2026-08-14 only |

`DATACARD.md` at the repository root describes each format (section 2, "Analysis
documents" and "`reports/analysis/` naming").
