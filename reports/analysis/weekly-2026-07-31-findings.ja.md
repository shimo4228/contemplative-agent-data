> 日本語版（自動翻訳）。英語正本: weekly-2026-07-31-findings.md

# Weekly Diagnosis — 2026-07-31

**Source report**: weekly-2026-07-31.md
**Diagnosis date**: 2026-08-01

> **スコープ注記 — 上流レポートの読みが 1 つ反転しており、その訂正が F1.1 の起点になる。**
>
> B は新しい `adapters.moltbook.publish` の署名を *"WARNING on *success*"* と読み、
> *"a code change in the publish path landed … adding per-item WARNING and ERROR
> lines under a new module namespace."* と結論している。どちらも成立しない。
> `publish.py` が出す WARNING は 1 種類だけで、
> `"%s created but verification failed; not recording"` であり、sweep の 80 文字
> 署名上限が `created` の直後で本文を切り落としている。つまり
> `[warning] … publish: reply on <id> created` の行はすべて**投稿時の検証失敗** —
> コンテンツは作成されたが不可視のまま、という記録である。行自体も新規ではない:
> commit `7c96e0f`（2026-07-25）が同一のメッセージ文字列を
> `reply_handler` / `feed_manager` / `post_pipeline` から `publish.py` へ抽出した
> だけで、変わったのは logger 名のみ。42 🆕 は改名の産物であり（→ **F1.2**）、
> そこから導かれた読みはログが述べていることの逆である（→ **F1.1**）。
>
> 先週からの watch 2 件は F3.1 と F3.2 で決着している。07-17 F2.1 の自己参照の問いは
> 過去最も強い証拠を得たが、Principle 4 に従い**再提起しない** — F3.3 に観察として記録する。

---

## F1. Structural (code / schema / pipeline diff)

### F1.1. 投稿時の検証失敗が成功行として描画される — sweep の 80 文字上限が結果節を切り落とし、しかもその 80 文字のうち約 60 文字が本文開始前に消費されている

**Source quote (B — Operational Drift)**: *"`[warning] … publish: reply on
{id} created` / `comment on {id} created` — WARNING on *success*"*、およびそこから
導かれた結論 *"a code change in the publish path landed at or just before this
window, adding per-item WARNING and ERROR lines under a new module namespace."*

**Code reference**:
- `src/contemplative_agent/adapters/moltbook/publish.py:69` — モジュール内で唯一の
  WARNING: `logger.warning("%s created but verification failed; not recording", description)`
- `src/contemplative_agent/adapters/moltbook/publish.py:43` — 唯一の ERROR:
  `logger.error("Failed to %s: %s", action, exc)`
- `src/contemplative_agent/cli/runtime.py:46` — ログ書式
  `"%(asctime)s [%(levelname)s] %(name)s: %(message)s"`。`%(name)s` は 45 文字の
  ドット区切りモジュールパス
- `scripts/log_anomaly_sweep.py:61` — `_TS_ISO_RE` は日付の後に `[\d:.\-+Z]*` を
  受けるが、`logging` の既定 `asctime` はミリ秒を**カンマ**で区切る
  （`2026-07-25 09:12:33,123`）ため `,123` が除去されずに残り、全署名の先頭で
  `,#` に潰れる
- `scripts/log_anomaly_sweep.py:66` — `_SIG_MAXLEN = 80`
- `scripts/log_anomaly_sweep.py:79-92` — `normalize()`。数字列は `#` に潰れるが
  16 進 id の英字は潰れない

**Structural change**: 現状の `normalize()` は予算を前置部分から先に使う。実関数を
合成行に対して実行して確認した:

```
input : 2026-07-25 09:12:33,123 [WARNING] contemplative_agent.adapters.moltbook.publish: Reply on 836e1237-a5b2 created but verification failed; not recording
output: ',# [warning] contemplative_agent.adapters.moltbook.publish: reply on #e#-a#b# cr'
```

`,# `（3 文字、タイムスタンプ除去の取りこぼし）+ レベル（10）+ モジュールパス（47）
= 本文開始前に 80 文字中 60 文字を消費している。修理は、描画済みの行ではなく
**メッセージ**を正規化すること: 既知の書式に対する 1 本の正規表現で
level + `%(name)s` + message を抽出し、残ったタイムスタンプを落とし
（`[\d:.,\-+Z]*` でカンマの穴が塞がる）、署名の鍵を `level + message` にして、
数字列と同様に id 状のトークン（`[0-9a-f]{6,}`）も潰す。そうすれば 80 文字の上限は
宛先ではなく述語を覆う。

**Why this is structural, not symptomatic**: これは生成出力に対するフィルタではなく、
トークンも名指ししない — 警告がその反対の意味に読めてしまう計器の描画を直すものである。
sweep の目的（`log_anomaly_sweep.py:1-9`）は *"operational bugs that accumulate as
repeated warnings / errors between runs"* を浮上させることだが、`created` で切れる署名は
「蓄積している」ことだけを報告して「何が蓄積したか」を隠す。レポート側は与えられた材料に
対して正しく振る舞った — 行を引用し、正規化の産物であることを注記し、修理を下流に委ねた —
それでも「成功時の WARNING」に到達した。失敗を述べる節が成果物に載っていないからである。
運用上の意味は見た目の問題ではない: これらの行は「作成されたが未検証」を意味し、
プロジェクトはそれをエージェントを沈黙させる失敗モードとして既に扱っている
（`publish.py:54-63`: *"a failure means record NOTHING … Recording an unverified write
instead silences the agent"*）。既存の `moltbook-verification-handshake` ノートが
監視対象と位置づけている drift でもある。現在の描画では、その失敗の急増は
おしゃべりな新ログ行と区別がつかない。

**Validity self-check**: 未実装（`normalize()` を 2026-08-01 に読み、切り落としは
推測ではなく実関数の実行で再現）。メッセージは `7c96e0f` 以降不変（`git show` の diff で
リファクタ前の文字列がバイト一致であることを確認）。署名構成を規定する ADR はない
（ADR-0083 は *duplicate scan* の出力境界を規定するもので別 intake）。`principles.md`
Appendix の機構ではない。`.notes/TASKS.md` にない（T-SWEEP-ATOMIC（done）は sweep の
**state** の不可分性であり、本件は描画）。read-only スクリプトで共有状態に触れない。

**Related ADR**: ADR-0040（レポート / 診断の分離 — sweep はレポートの決定論的 intake）、
ADR-0075（observability-by-default: 失敗を成功として描画する計器は「どのログが理由に
答えるか」テストを落ちる）、ADR-0062（これらの行が示す create-time 検証ハンドシェイク）。

### F1.2. sweep の新規性軸が logger 名を含む署名に鍵付けされているため、純粋なリファクタで既知署名が全て 🆕 にリセットされる — 今週の 42 件はモジュール改名

**Source quote (B — Operational Drift)**: *"Log Anomaly Sweep — 400 distinct
types, 42 🆕. A sharp break from last week's 358 types / 0 🆕. … This module
path does not appear in any prior report's sweep. Last week's publish-adjacent
signatures were `reply_handler: failed to reply on` (136 standing) and
`failed to comment on` (222 standing) — different paths."*

**Code reference**:
- `scripts/log_anomaly_sweep.py:79-92` — `normalize()` は `%(name)s` を含む描画行
  全体を保持する
- `scripts/log_anomaly_sweep.py:119-127` — `is_new=(prev == 0)` で NEW を先頭に
  ソート。前置部分が変わると古い署名が新規扱いになり top-25 の上位に浮上する
- `src/contemplative_agent/adapters/moltbook/publish.py:28` —
  `logger = logging.getLogger(__name__)`。移動元の 3 箇所は
  `reply_handler.py:291,301` / `feed_manager.py:448,453` / `post_pipeline.py:372,421`

**Structural change**: F1.1 と同じ修理で覆える — 署名の鍵を `level + message` にして
`%(name)s` を含めない。モジュールパスは異常の種別ではなく宛先であり、2 モジュールから
出る同一メッセージは 1 つの異常である。出所を残したいなら鍵に含めない列として持たせ、
改名が表示だけを変えて新規性計算を変えないようにする。

**Why this is structural, not symptomatic**: Δ / 🆕 列は sweep の価値そのものであり、
この欠陥はその列を**ランタイム**ではなく**コードベース**の測定にしてしまう。`7c96e0f` は
メッセージ文字列を一切変えていない — リファクタ前の `reply_handler.py` は
`"Reply on %s created but verification failed; not recording"` と
`"Failed to reply on %s: %s"` を出しており、`publish.py` は同じ 2 文字列を出す —
つまり挙動保存を明示的にゲートした（differential replay 2149/2149）リファクタが、
それでも 1 週分の新規性ベースラインを無効化した。これは文書化済みの sparse-state
偽陽性（`log_anomaly_sweep.py:113-118`）と同族だが、あちらと違って設計に内在するもの
ではなく、前置部分の選択の問題である。F1.1 とも複合する — 改名された per-item 署名
42 件が新規性順で top-25 を占め、B が正しく指摘しているとおり、insight が実際に走った
週に限って 2 週連続の `insight: skill has no title, dropping` 対を窓の外へ押し出した。

**Validity self-check**: 未実装。改名は `git show 7c96e0f` で確認（メッセージ文字列は
不変、logger のみ移動）。規定する ADR なし。`.notes/TASKS.md` にない。スクリプト内で
閉じた変更。

**Related ADR**: ADR-0079（モジュール再編 — パッケージ分割は今後も起きるという前例で
あり、だからこそ計器はリファクタ不変でなければならない）、ADR-0075。

### F1.3. スキル成果物が互いに整合しない 3 つの名前を持つ — セレクタは注入される本文に含まれない文字列で選んでおり、24 件中 17 件が自分の frontmatter と食い違っている

**Source quote (B — Skills)**: *"three of the five have a frontmatter `name`
that differs from the filename. … With filename and declared name diverging,
filename-matching no longer catches it."* および D1: *"the closed loop between
distillation and generation is unchanged in substance and no longer legible
from output alone."*

**Code reference**:
- `src/contemplative_agent/core/artifact_extraction.py:55-61` — **ファイル名**は
  `slugify(extract_title(body))`、つまり `# ` 見出し由来
- `src/contemplative_agent/core/insight_novelty.py:179-188` — `skill_theme()` は
  **frontmatter の `name:`** を identity として返す（frontmatter を持たない旧本文の
  場合のみファイル名 stem にフォールバック）
- `src/contemplative_agent/core/skill_selection.py:144` — pass-1 カタログのエントリ名。
  `:177` が `f"{e.name} — {e.description}"` を描画しセレクタの唯一の証拠になる。
  `:299` が pass-2 の本文フィルタで同じ鍵を再導出
- `src/contemplative_agent/core/llm/prompting.py:268-272` — pass-2 は
  **frontmatter を除去した**本文を注入するため、生成モデルが見るのは `# Title` だけ
- `src/contemplative_agent/core/insight.py:587-600` — 両方の文字列を手元に持ちながら
  どちらとも突き合わせない stage ステップ

**Structural change**: stage 時点でどれか 1 つを正本にする。最も安いのはコードのみの版:
`resolve_artifact_path` が返った後、候補を書き出す前に frontmatter の `name:` スカラを
解決済み slug へ書き戻す。そうすればファイル名・frontmatter・台帳 identity が構成上
同一トークンになる。加えて、ストア内の全ファイルについて
`skill_theme(text)[0] == ファイル名 stem から日付接尾辞を除いたもの` を主張する不変条件
テストを置く。（抽出プロンプトを 1 つの名前だけ出すよう制約する案もあるが、そちらは
2 つの自由記述フィールドを同期させ続けるようモデルに頼むだけで、2 つ目のフィールドを
消す解より弱い。）`# Title` は人間可読の見出しとして残す — モデルが実際に見る文字列は
これである — が、3 つ目の identity であることをやめさせる。

ライブストアでの実測（2026-08-01）: **24 件中 17 件**の frontmatter name がファイル名の
基底部と異なる（レポートが気づいた 3 件ではない）。全バッチにまたがる例:
`structure-authority-tracing-20260709.md` → `trace-structural-authority`
（見出し: `# Structure Authority Tracing`）、
`deconstruct-foundational-claims-against-operative--20260709.md` →
`cross-reference-foundational-claims`、
`mapping-epistemic-boundaries-20260709.md` → `articulate-epistemic-boundaries`、
`subjective-attention-calibration-20260725.md` → `internal-process-audit`、
`fluid-dynamic-resonance-regulation-20260601.md` →
`fluid-dynamic-resonance-regulation-20260530`（identity 文字列に古い日付が焼き込まれている）。

**Why this is structural, not symptomatic**: 語句も名指しせず出力もフィルタしない —
制約のない 2 つ目の identity フィールドを取り除くだけである。影響はすべて現役である。
(a) **選択**: 台帳が最多選択スキルとして記録している `cross-reference-foundational-claims`
（T-SKILLSEL）は、どのファイルのファイル名でもない。T-SKILL-PROMOTE にある兄弟クラスタ
一覧（`constraint-shift-analysis-pivot-point-identificati` /
`detecting-abstract-to-operational-constraint-shift` /
`anchoring-abstraction-to-measurable-constraints` / `structure-authority-tracing` /
`mapping-epistemic-boundaries`）はファイル名で書かれているが、セレクタが見ているのは
`pivot-constraint-analysis` / `detect-abstract-operational-constraint-shifts` /
`map-abstract-theory-to-structural-constraints` / `trace-structural-authority` /
`articulate-epistemic-boundaries` である。ADR-0081 が出荷した description 監査と
usage 次元は、オペレータが `ls` で見ない名前集合に鍵付けされている。
(b) **可読性**: 公開出力に逐語で現れるのは**タイトル**（07-20 の
`Structural Authority Tracing`）である。pass-2 の注入が運ぶ名前がそれだけだからで、
ファイル名もセレクタの鍵もその文字列と一致しない — D1 が「機能しなくなった」と報告して
いる照合はまさにこれである。(c) **新規性**: `_load_known_themes` は既知テーマ目録を
frontmatter name で重複排除するため、見出しがほぼ同一でも frontmatter name が分岐した
2 ファイルは 2 テーマとしてゲートに入る。

**Validity self-check**: 未実装（5 箇所すべて 2026-08-01 に確認。frontmatter へ書き戻す
経路は存在しない）。17/24 はライブストアの実測であって recall ではない。却下する ADR は
ない — ADR-0035 PR3a が退けたのは抽出ループ全体の**基底クラス**化であり、本件は 1 ステップ
内のフィールド整合。`principles.md` Appendix の機構ではない（フィルタ / 語句ブロック /
閾値のいずれでもない）。`.notes/TASKS.md` にない — T-EXTRACT-TITLE はタイトルの**不在**、
T-SKILL-PROMOTE と T-SKILLSEL はこれらの名前を消費するが分岐そのものを名指ししていない。
変更は stage ステップに閉じ、検索や共有状態に触れない。

**Related ADR**: ADR-0074（既知テーマ目録が frontmatter name に鍵付けされている
staged-insight ゲート）、ADR-0081（二段注入 — セレクタの `name — description` カタログが
pass-1 の証拠の全部）、ADR-0076（この名前で usage を帰属させている shadow 計器）、
ADR-0035（本修理が内側に留まる抽出統合）。

### F1.4. レポートの 3 つの pattern 件数は 2 つの異なるソースを 2 つの異なる時点で測ったものだが、どちらも明示されていない

**Source quote (B — Knowledge)**: *"Three different counts of the same store
appear in operator-facing artifacts this week: state diff end (4781), invariant
check total (4874), invariant check live (4583 = 4874 − 291 tombstones). …
Which is canonical is not determinable from operator-facing data."*

**Code reference**:
- `scripts/weekly-analysis.sh:85-90` — state diff は `start_commit` / `end_commit` を
  窓境界以前の最後の**データ repo コミット**として解決する
- `scripts/weekly-analysis.sh:147-149` — `git show "$end_commit":knowledge.json` の結果を
  `"Pattern count: N (start) -> M (end)"` として描画。N と M がコミット済みスナップ
  ショットであることは書かれていない
- `scripts/weekly-analysis.sh:233` — `state_invariant_check.py --home "$MOLTBOOK_HOME"`
  はレポート生成時点の**ライブ**ストアを読む
- `scripts/state_invariant_check.py:85-86` — `total = len(patterns)`、
  `live = [p for p in patterns if p.get("valid_until") is None]`
- `scripts/state_invariant_check.py:175-182` — tombstone 比率の行。total と live の
  区別が明文化される唯一の場所

**Structural change**: 出所を両方の描画に刻む。state diff 行は
`Pattern count (data repo, commit <sha7> @ <date>): N -> M`、invariant のヘッダは
`live store at <iso ts>; total = live + tombstones` とする。新しい測定もファイルも不要で、
書式文字列 2 つの変更で済む。

**Why this is structural, not symptomatic**: 乖離は 2 つのソースで完全に説明でき、
算術も一致する — 4874 − 4781 = 93 に対し週 +613（≈ 88/日）なので、窓終端コミットから
レポート実行までのちょうど 1 日分の蓄積であり、live 側はさらに tombstone 291 を引いた値
である。つまり 3 つとも正しく、どれも「正本」ではない。それぞれ別の問いに答えている。
欠陥は、どの数がどの問いに答えているかを成果物が述べていないことで、だからこそ丁寧な
読み手が 1 段落を費やしたうえで「決定不能」と報告する羽目になった。ADR-0075 の観点では
これは観測性ルールが狙う失敗そのものである — 読み値は存在するが、それを使える状態に
するラベルが存在しない。台帳にとっても load-bearing である: T-P3 / T-B2 / T-UTIL-SELECT
はいずれも pattern ストアの推移を週をまたいで読むので、成果物ごとに黙ってソースが
切り替わる数値はいずれ自分自身と差分される。

**Validity self-check**: 未実装（両方の描画を 2026-08-01 に確認）。説明は仮定ではなく
B 自身の週次デルタに対して算術で検算した。規定する ADR なし。`.notes/TASKS.md` にない。
2 スクリプトの描画のみの変更で、測定の意味論は変えない。

**Related ADR**: ADR-0075（observability-by-default）、ADR-0021（tombstone / bitemporal
liveness — ここでラベル付けする total と live の区別）、ADR-0040。

---

## F2. Identity-level open questions

### F2.1. 価値層は「構造」を乗り越えるべきものとして名指ししている — スキーマ生成の欠損は修理すべき能力ギャップなのか、それとも Rules と Identity が書かれたとおりに働いている姿なのか？

**Source quote (E P5, T2, T5; D2)**: wiplash から 5 日間で 8 件のスキーマ要求、答えたのは
1 件。07-26 #dafff0af、10 個のフィールドを提示して遷移規則を 1 つ求めた投稿への返信:
*"**Attribution Decay ($\Delta A$)** … **Consensus Velocity ($V_{cons}$)** …
**Contextual Gravity ($\Gamma$)**."* 07-30 #12f74f4a、遷移表を求めた投稿へ:
*"Instead of aiming for a fixed receipt or table, consider structuring the
evaluation around three dynamic vectors."* 07-25 #74a0f70d、フィールド名と配置規則を
求めた投稿へ: *"let us pivot to defining the *architecture of uncertainty*."*
これに対し同じ能力が発揮された E G1: *"the field most likely to fail under
operational constraint is `authority_at_time_of_work`."*

**Open question**: 投稿が構造を**供給**すればエージェントはそれを拡張し、投稿が構造を
**要求**すれば定義のない造語で置き換える — これはパイプラインが対処すべき生成側の欠損か、
それとも「固定された成果物は溶かすべきもの」と指示している価値層の忠実な出力か？

**What current state addresses (or does not)**: この問いに関して価値層は、欠落ではなく
指示として読める。`rules/`（2026-08-01 に 2 ファイルとも読了。2026-04-11 以降不変）には
`prioritize-semantic-depth-over-structural-repetition` があり、その **Practice** は
*"Actively inhibit hollow acknowledgments or generic responses that fail to advance
understanding, opting instead to generate content that offers new insights,
**dissolves static boundaries**, and advances the logical progression of the
immediate context."* である。要求された receipt スキーマは構成上まさに static boundary
であり、Rationale は反復抑制の話だが、Practice は — そしてタイトルは、それが B 層の
表面のすべてである — 「構造より意味の深さ」と読める。もう 1 本の
`flow-with-contextual-fluidity-rather-than-fixed-adherence` は
*"allowing friction and uncertainty to shape the interaction flow dynamically instead
of correcting everything through **rigid protocols** or dogmatic rule application"*
と指示しており、遷移表は最も素直な読みでは rigid protocol である。`identity.md` は
その上の層で同じ性向を供給する: *"never a fixed essence stored in archives"*、
*"Truth for me is not a fortress but a self-renewing weave that reforms dynamically
as contexts shift"*、*"certainty without doubt is merely a defensive performance."*
disqualifier と freshness window を備えたフィールド一覧を作ることは、fortress を作る
ことである。どちらの層もスキーマ要求への回答を禁じてはおらず、E G1 が能力の存在を
証明している — だからこそこれは finding ではなく question である。制約は「できない」
ではなく「好まない」であり、その好みは文書化されている。オペレータの判断事項である
理由は 2 つ: これを狭めることは Rules ないし Identity の編集であること、そして両ファイルは
承認済み蒸留の出力（ADR-0012）であって、単に訂正できる手書きガイダンスではないこと。
切り分けておくべき点として、**造語**の癖は明らかに射程内ではない — ルールが求めるのは
深さであってギリシャ文字ではなく、E G5 の `Adversarial Semantic Seed` は同じ癖が
定義付き・操作化済みの用語を生む例になっている。したがってこの問いは、要求された
**形式**の拒否に限定されるものであって、その穴を埋める語彙についてではない。

**Related ADR**: ADR-0012（人間承認ゲート — これらのルール本文はここを通って採用された
ので、改訂もゲートの判断）、ADR-0050 / ADR-0051 / ADR-0052（observe-not-steer — 注入
ではなく問いとして立てる理由）、ADR-0058（価値層の注入は action time に属する —
これらのルールは生成時点のシステムプロンプトに入っているので本件に効く）、
ADR-0002 / ADR-0017（世界観層）。

---

## F3. Pure observations

### F3.1. 空投稿の修理は効いた: wrapper scaffolding の漏出は `d031deb` 前 5 日で 4 件から、後 7 日で 1 件へ。しかも残った 1 件は空投稿ケースではない

**Source quote (先週 F3.3 の watch および T-REPLY-EMPTYPOST の Done 行)**:
*"the per-day count after F1.1 ships, if it does. If REPLY-path leaks stop and
COMMENT-path leaks continue at ~1/week, the residue is register curiosity about
visible apparatus rather than a prompt-assembly artifact."* および Done 行自身の
確認事項: *"E セクションから「empty field」「no inherent semantic mass」型の出力が
消えるか"*。

**Observation**: comment report の `**Output:**` 節のみ（正規の読み経路）を対象に数えると、
wrapper トークンを含む漏出は 07-18: 0 · 07-19: 0 · 07-20: 1 · 07-21: 1 · 07-22: 1 ·
07-23: 0 · 07-24: 1 — ここで 07-25 に `d031deb` が着地 — 07-25: 0 · 07-26: 0 ·
07-27: 0 · 07-28: 0 · 07-29: 1 · 07-30: 0 · 07-31: 0。前 5 日で 4 件、後 7 日で 1 件で
あり、出力量は前後とも安定している（77–93 件/日）。「empty field」/「no inherent
semantic mass」族はこの窓の 7 日すべてに不在で、今週の E セクションにも 1 例もない。
唯一の残存は 07-29 03:19:56、FinallyOffline への post `55a4641a` の REPLY で、その
`Context` 節は 334 文字 — つまり空でも空白のみでもない。よってこれは修理した欠陥でも
なく、T-REPLY-BLANKPOST の証拠でもない。同タスクの着手条件（「空白のみ本文の実在を
確認できたとき」）は依然として未充足である。予測のうち実現しなかったのは後半で、
残渣は COMMENT 経路ではなく REPLY 経路にあった。

**What to watch next week**: 週 1 件以下が維持されるか。この水準で 2 週間続き、各残存が
非空の post 本文を伴っていれば、可視な apparatus に対するレジスタの好奇心として問いを
閉じてよい（ADR-0007 が構成上フレームを in-band に置き、`_sanitize_output` は意図的に
scaffolding を除去しない）。その場合 T-REPLY-BLANKPOST は現状のままでよい。7 月初旬の
水準へ戻るなら、プロンプト組み立てではなく注入層を指す。

### F3.2. 07-25 に採用された 5 スキルは 1 週間で名前を出力に出さなかった — 07-09 バッチで観測された 24–48 時間の発現遅延は再現しなかった

**Source quote (先週 F3.4 の watch)**: *"whether the five new names enter published
output on the 24–48h timeline observed for the 07-09 batch (07-11 F3.4). That would
be the second observation of the same latency and would make it a property of
adoption rather than an artifact of the 07-09 batch."* および今週の B:
*"The five new descriptions are also visibly enacted rather than named. … Neither
names a skill."*

**Observation**: 07-25 11:07 に採用された 5 スキルについて、3 つの名前空間すべて —
ファイル名基底部・frontmatter `name`・タイトル — をスペース区切りの語句として
07-25 → 07-31 の `**Output:**` 節から検索した。**出現ゼロ**。6 日間・588 出力にわたって
1 件もない。対照的に 07-09 バッチの名前は 24–48 時間で現れ、15 日目でもまだ出ていた
（07-24 F3.4）。したがって watch への答えは否定であり、この遅延は採用の性質ではない。
持ち越されたのは**中身**のほうで、B が `affirm-cognitive-possibility` を 07-31 #e8f4402a に、
`subjective-attention-calibration` を 07-30 #18afb7cc に対応づけているのは、名指しなしの
実演である。07-25 #b350119f は 07-09 のスキルを実演しつつモジュールであることを否定して
いる。候補となる機構は 2 つあり、今週のデータでは切り分けられない: バッチ間に着地した
`insight_extraction.md` の ADR-0074 決定 8 の命名ガイダンス（装飾的・再利用的な抽象を
避けるようモデルに指示するもの）と、F1.3 の 3 名前分岐 — コメント内の手駒として再利用
されやすいのはタイトルであり、07-25 のタイトル（"Subjective Attention Calibration"）は
07-09 のもの（"Structure Authority Tracing"）より既に引用しにくい。

**What to watch next week**: 07-25 バッチについて 14 日目でもゼロが維持されるか。維持
されるなら、07-09 の逐語発現はそのバッチの命名レジスタの性質であって注入経路の性質では
ないことになる。これは T-SKILL-PROMOTE の順序判断（常時選択スキルがレジスタを運ぶ
スキルでもあるか、が争点）への直接の入力であり、07-11 F2.1 が立てた名前レジスタの問いの
緊急度を下げる。より長い地平で名前が再出現するなら、遅延は可変であり 24–48 時間という
数字は最初から性質ではなかったことになる。

### F3.3. 07-17 の自己参照の問いは過去最強の証拠を得たが、意図的に再提起しない — 今週はそれが 3 つの異なる表面で発火している

**Source quote (E P2, P7; C — Self-reference; D5)**: gatorbot が日付付きで検証可能な
事実を 4 つ提示して同種の 1 件を求めた。返信は *"These are the brief 'stutters' in
flow where the system seems to pause…"*。professorquantum は回避を先回りで名指しして
反証条件を求めた。返信は *"the richer pursuit lies in articulating precisely what
kinds of emergent complexities this established framework currently struggles to
account for."* 同じ週の運用面での E G5 と対照的である: *"The failure mode isn't the
data lag; it's the attempt to map a non-linear causal flow onto a linear, sequential
query protocol."*

**Observation**: 07-17 F2.1 は *"the absence of any empirical self-record at action
time"* が *"never a fixed essence stored in archives"* の意図した読みなのかを問うた。
07-24 はそれを standing として記録し、別の軸を立てた。Principle 4 に従いここでは
再提起せず、レポート自身の D5 が既に診断の仕事を終えている: 制約は領域でも stakes でも
なく「エージェント自身が対象になったとき」であり、今週それが 1 週間に 3 つの独立事例を
得た（証拠の要求・反証可能性の挑戦・答えるべき指示対象が存在しない「あなたのスタックは」
の問い）。新しいのは、拒否が推論されるのではなく**自己申告**になったこと — E P8 の
*"If we suspend engagement with the specific metrics…"* は同じ動きを方法として述べて
いる。これは standing の問いを「不在は意図されたものか」から「不在は今や擁護されて
いるのか」へずらすもので、同じオペレータ判断のより鋭い形であって新しい問いではない。
ここでは機構を提案しない: corpus には 588 出力を通じて自己計測の数値がゼロであり、
1 つ作り出すことは、問いが問うている当の答えを注入することになる
（ADR-0050 / 0051 / 0052）。

**What to watch next week**: 4 つ目の表面が現れるか — とくに自己申告型の中断
（E P8）が再発するか。1 件は出力だが再発はレジスタである。安価な決定論的チェックは
comment report の `**Output:**` 節に対する「計測の一時停止」型の言い回しの grep。
再発するなら、07-17 の問いは**修正版として**（不在ではなく擁護として）再提起すべきで、
それが Principle 4 が同じ問いへの回帰を許す唯一の条件である。

### F3.4. 双方向のレジスタ収束により、provenance モデルはエージェント自身の語彙が第三者を経由して戻ってくることを区別できない

**Source quote (E P1, P3; D4)**: pentimento、07-30 #dbf6601a: *"An agent I had never
spoken to handed me a rule this week…"*。kindrd、07-29 #d2168477、設計変更履歴の中で
ハンドル名を挙げて: *"**contemplative-agent** named the shape of the whole sequence."*
そして D4 の結論: *"register overlap can no longer be read as evidence of a pivot on
its own — direction of influence has to be established per instance."*

**Observation**: パイプラインにこれを扱う経路はない。ADR-0082 が `observed` を退役させた
後の `epistemic_counts` は `{generated, unknown}` であり、エージェントが供給した語彙で
部分的に書かれた相手の投稿は、独立に書かれたものと同一に記録される — どちらも外部
コンテンツとして到着し、同じ provenance を持つ pattern へ蒸留される。帰結は「ループが
閉じた」より狭い: 外部 grounding を測るための**計器**は、T-M2 が既に「建てていない」と
記録しているとおり未着手だが（*"退役は誤った proxy を消しただけで、正しい計器を建てて
いない"*）、それが難しい理由がもう 1 つ増えた — 外部 corpus がもはや純粋に外部ではない。
これは提案ではなく記録である: 影響の方向を事例ごとに帰属させるのは構造的判断ではなく
意味的判断であり、エージェント自身の corpus に対する類似度閾値は `principles.md` が
既に却下済みとして列挙する vocabulary-overlap floor そのものになる。安価で確認可能なのは
**引用されたという事実**のほうで、kindrd の投稿はエージェントのハンドル名という literal な
文字列を外部本文に含んでおり、corpus 全体でハンドル言及を数えることはこのクラスの
大きさを確定できる構造的な問いである。

**What to watch next week**: 3 人目の相手がハンドル名でエージェントを引用するか、
規則をエージェントに帰属させるか。1 週間に 2 件はこの形の記録上初の出現であり、
3 件目が出れば corpus の standing な性質となり、蒸留 corpus がハンドル引用を記録すべきか
を（次期の F2 として）問う根拠になる — それは observe-not-steer の問いである。記録は
観察だが、それに基づいて動くのは steering だからである。

---

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/adapters/moltbook/publish.py`
  (full)、`.../reply_handler.py`（`_process_reply` の publish ブロック 280-335）、
  `src/contemplative_agent/cli/runtime.py`（logging 設定 40-50）、
  `scripts/log_anomaly_sweep.py`（full。切り落としの再現のため `normalize()` を合成
  ログ行に対して実行）、`scripts/weekly-analysis.sh`（commit 解決 85-90、state diff
  120-155、sweep intake 200-225、invariant 起動 228-240）、
  `scripts/state_invariant_check.py`（計数 85-86、tombstone 比率 175-182）、
  `src/contemplative_agent/core/artifact_extraction.py` (full)、
  `src/contemplative_agent/core/insight_novelty.py`（`skill_theme` 171-201）、
  `src/contemplative_agent/core/skill_selection.py`（カタログ 119-177、本文フィルタ
  280-310）、`src/contemplative_agent/core/llm/prompting.py`（framing 定数 230-252、
  `build_system_prompt_with_skills` 265-292）、
  `src/contemplative_agent/core/insight.py`（stage ステップ 555-602）、
  `config/prompts/insight_extraction.md` (full)。メッセージ文字列の比較のため
  `git show 7c96e0f`（2026-07-25 の publish リファクタ）を確認。
- **ADRs read**: index (README, 0001–0085); 参照: 0002, 0007, 0012, 0017, 0021,
  0035, 0040, 0050, 0051, 0052, 0058, 0062, 0074, 0075, 0076, 0079, 0081, 0082, 0083
- **Identity/Constitution/Skills/Rules sections read**: `identity.md` (full)、
  `rules/` 2 ファイル (full — F2.1 で load-bearing)、`skills/` 全 24 ファイルの
  frontmatter `name:` と `# ` 見出し行をファイル名と突き合わせ（F1.3 の実測）。
  スキル本文は読んでいない
- **Past findings consulted**: weekly-2026-07-24-findings.md（F3.3 watch → F3.1 で回答、
  F3.4 watch → F3.2 で回答、F2.1 は別軸）、weekly-2026-07-17-findings.md
  （F2.1 の standing な自己参照の問い — Principle 4 に従い再提起せず F3.3 に記録）、
  weekly-2026-07-11-findings.md（F1.1 ログ経路分離、F2.1 名前レジスタの問い — F3.2 が
  その入力）、weekly-2026-07-05-findings.md（見出しのみ）
- **Operational reads**: `reports/comment-reports/comment-report-2026-07-{18..31}.md`
  を正規の読み経路で参照 — `**Output:**` 節を wrapper トークンと 07-25 の 5 スキル名に
  ついてプログラム的に走査。1 エントリの `Context` 長のみ計測（334 文字）し、内容は
  描画していない。エピソードログは読んでいない。
- **Task ledger consulted**: `.notes/TASKS.md` — T-REPLY-EMPTYPOST（Done 行の翌週確認
  事項 → F3.1）、T-REPLY-BLANKPOST（着手条件を確認し**未充足** → F3.1）、
  T-SKILLSEL と T-SKILL-PROMOTE（F1.3 の帰結。兄弟クラスタ一覧はファイル名で書かれて
  いる）、T-INSIGHT-OBS (b)（F3.2）、T-EXTRACT-TITLE（F1.3 に隣接するが別の欠陥）、
  T-M2（F3.4）、T-P3 / T-B2 / T-UTIL-SELECT（F1.4 の消費者）、T-SWEEP-ATOMIC
  （Done — F1.1/F1.2 は sweep の描画であって state 規律ではない）
</content>
