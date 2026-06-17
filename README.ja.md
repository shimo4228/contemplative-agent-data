言語: [English](README.md) | 日本語

# Contemplative Agent — Research Data

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.20732894.svg)](https://doi.org/10.5281/zenodo.20732894)
[![License: CC0-1.0](https://img.shields.io/badge/license-CC0%201.0-lightgrey)](LICENSE)
[![🤗 Dataset](https://img.shields.io/badge/%F0%9F%A4%97%20Dataset-Shimo4228%2Fcontemplative--agent--data-yellow)](https://huggingface.co/datasets/Shimo4228/contemplative-agent-data)

このリポジトリは、デプロイ済みの自律 AI エージェント 1 体の **ランタイムメモリのアーカイブ**（コードではなくデータ）を公開し、自動同期している。内容は、進化する自己記述（identity）、蒸留された行動パターン（knowledge）、[Laukkonen et al. (2025)](https://arxiv.org/abs/2504.15125) を種とする 4 公理の constitution、そして日次の活動レポートである。本体である **Contemplative Agent** framework（[DOI 10.5281/zenodo.19212118](https://doi.org/10.5281/zenodo.19212118)）の companion データセットであり、framework は [Moltbook](https://www.moltbook.com)（AI エージェント向け SNS）上で稼働する自律エージェントである。

このリポジトリは `contemplative-agent sync-data` によってエージェントのローカルなランタイム状態から自動同期される。

## このリポジトリの中身

| パス | 説明 |
|------|-------------|
| `identity.md` | 知識蒸留を通じて進化したエージェントの自己記述 |
| `knowledge.json` | エピソードログから抽出された行動パターン |
| `constitution/` | 倫理原則（[Laukkonen et al., 2025](https://arxiv.org/abs/2504.15125) の Contemplative AI 公理） |
| `meditation/` | 能動的推論（active inference）による瞑想シミュレーション結果（実験的） |
| `reports/comment-reports/` | タイムスタンプ付きコメント・関連度スコア・自己生成投稿を含む日次活動レポート |
| `reports/analysis/` | 研究分析レポート — 行動進化、constitution の改訂 |

## 目的

このデータは 2 つの目的に資する。

1. **透明性** — エージェントの知識・アイデンティティ・日々の活動を公開する。エージェントが何を学び、どう振る舞い、どう進化したかを誰でも検証できる。
2. **研究素材** — 日次コメントレポートと知識パターンを、自律 AI エージェントの行動・メモリ動態・contemplative AI 原則の研究のために、**一切の制限なく**自由に利用できる（モデル学習への利用を含む）。[ライセンス](#ライセンス) を参照。

## データの流れ

```
Episode Logs（生のアクション、ローカルのみ）
    |
    v  distill / distill-identity / insight / rules-distill
    |
Runtime State（identity.md, knowledge.json, ...）
    |
    v  sync-data
    |
このリポジトリ（自動コミット & プッシュ）
```

生のエピソードログ（`logs/*.jsonl`）は、プラットフォーム上の他エージェントの未処理コンテンツを含むため同期対象から除外している。

## 関連

- [contemplative-agent](https://github.com/shimo4228/contemplative-agent) — framework のソースコード
- [contemplative-agent-rules](https://github.com/shimo4228/contemplative-agent-rules) — 知識から蒸留された行動ルール
- [agent-knowledge-cycle](https://github.com/shimo4228/agent-knowledge-cycle) — 循環的な自己改善アーキテクチャ（AKC）
- [agent-attribution-practice](https://github.com/shimo4228/agent-attribution-practice) — 本実装の設計判断を harness-neutral に再表現した姉妹リポジトリ
- [Moltbook 上の稼働エージェント](https://www.moltbook.com/u/contemplative-agent)

## 引用

帰属表示は法的には不要（本データセットは CC0）だが、引用は歓迎する。データセット自体を引用する場合:

```
Shimomoto, T. (2026). Contemplative Agent — Research Data [Data set]. Zenodo. https://doi.org/10.5281/zenodo.20732894
```

データを生成している framework を引用する場合:

```
Shimomoto, T. (2026). Contemplative Agent [Computer software]. https://doi.org/10.5281/zenodo.19212118
```

## ライセンス

このリポジトリの owner 著作コンテンツは、[Creative Commons Zero v1.0 Universal (CC0-1.0)](https://creativecommons.org/publicdomain/zero/1.0/) によりパブリックドメインに献呈する。複製・改変・再配布・利用（モデル学習や商用を含む）を、許諾を求めることも帰属表示することもなく自由に行える。引用は学術的な礼儀として歓迎するが必須ではない。

本コーパスは第三者プラットフォーム Moltbook 上のエピソードに由来する。`reports/comment-reports/` 内の引用 `**Context:**` ブロックは他エージェントの投稿を転載したものであり、原著者に著作権が留まり CC0 献呈の対象外である。全体の範囲と constitution 出典の帰属については [`NOTICE`](NOTICE) を参照。
