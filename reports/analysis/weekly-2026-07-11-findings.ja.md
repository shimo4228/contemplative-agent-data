> 日本語版（自動翻訳）。英語正本: weekly-2026-07-11-findings.md

# Weekly Diagnosis — 2026-07-11

**Source report**: weekly-2026-07-11.md
**Diagnosis date**: 2026-07-12

> **Scope note.** 上流レポートからは見えない 4 つの事実が repo の記録とタスク台帳
> から確定でき、今週の支配的シグナルの大半は「**期間内に既に診断・修正済み**」と
> 読み直せる:
>
> 1. **新規 13 スキルは人間の承認ゲートを通過している。** 07-09 のランは ADR-0074
>    staged insight の初回手動実行で、632 パターン → 65 クラスタ → 54 staged →
>    `adopt-staged` による operator レビュー — **13 採用 / 41 却下**、audit 記録付き
>    （台帳 T-INSIGHT-BOOT、2026-07-09 完了）。レポートの「ロスの多いバッチの生存者」
>    という枠組みは不足で、13 件はロスの多いバッチ**かつ**個別の人間レビューの生存者である。
> 2. **前週の F1.1 は実装され、効いた。** learned-corpus の disposition framing は
>    2026-07-06 に出荷済み（`50b8070`）。D1 の観察 — 起動スキャフォールディングが
>    07-06 前半には現れ、07-09 までに Output から消えた — はこの修正が予定どおり
>    効き始めた姿である。
> 3. **コンテキスト予算の逼迫は期間中に発見・修正済み。** 🆕 `skipping llm call`
>    (+62) と 07-09 の自己投稿ゼロは、スキルストア成長（6 → 19 ファイル、~60 KB）が
>    推定入力を pre-flight guard の閾値越えに押し上げ、当時の guard が呼び出しを
>    *skip* していたことによる。`15ae37f`（2026-07-10）が skip → clamp に変更し、
>    `2f616eb` が adopt ゲートに system-prompt budget 計器を追加。自己投稿は
>    07-10（4 件）・07-11（4 件）に再開した。
> 4. **「fluid スキルのトリガーぼかし」は ADR-0048 の altitude clean パスである。**
>    `2f616eb` により stocktake clean の書き換えが in-place で着地するようになった。
>    "repeats every fixed interval" → "repeats in similar contexts" は
>    trigger-altitude の汎化が設計どおり動いたものであり、silent な破損ではない。
>
> 今週の F1 は 1 件: log チャネル衛生の欠落 — 生成本文の全文が、Log Anomaly Sweep が
> 読むのと同じ `*.log` ストリームへ INFO で流れており、それがエージェント自身の散文が
> 🆕 anomaly signature として浮上した経路である。

---

## F1. Structural (code / schema / pipeline diff)

### F1.1. 生成本文の全文が launchd log ストリームへ INFO で記録され、anomaly sweep がそれを読み込んでいる — コンテンツ記録と運用 log チャネルの混線

**Source quote (B — Log Anomaly Sweep)**: 🆕 (+1) *"the pivot point here feels
profoundly resonant: using the \*truncation\* itself as"* — 「エージェント自身が
生成した Output の断片…が log-anomaly ストリームに現れたもの、すなわち sweep が
読むチャネルへエージェントの散文が漏出している」; 加えて 🆕
`feed_manager: comment on …` / `reply_handler: reply on …` の WARN 行スプレー。

**Code reference**:
- `src/contemplative_agent/adapters/moltbook/feed_manager.py:467` — `logger.info(">> Comment on %s:\n%s", post_id[:12], comment)`
- `src/contemplative_agent/adapters/moltbook/reply_handler.py:348-350` — `logger.info(">> Reply to %s on %s:\n%s", …, reply)`
- `src/contemplative_agent/adapters/moltbook/post_pipeline.py:424` — `logger.info(">> New post [%s] (id=%s):\n%s", title, post_id, content)`
- Sweep の取り込み: `scripts/weekly-analysis.sh:174-186` — `*.log` の決定論的取り込み（`log_anomaly_sweep.py`）、行 signature を novelty でランク付け。

**Structural change**: INFO 記録に複数行の本文を埋め込むのをやめる。対話的な
可視性（tmux で眺める運用）のために logger プレフィックス付きの 1 行プレビューを
残し（例: `logger.info(">> Comment on %s: %d chars: %.80s…", post_id[:12],
len(comment), comment.replace("\n", " "))`）、全文は DEBUG でのみ出す。正本の
全文記録は既に存在し、変更しない: episode log の `content` フィールド（各呼び出し
箇所の直下で書かれている）と comment-reports の再生成
（`reports/comment-reports/`、公認の読み取り経路）。

**Why this is structural, not symptomatic**: 生成・公開されるものには一切触れず
（post-generation filter ではない、Principle 1）、token を名指ししない
（Principle 2）。ADR-0075 が既に別々の役割を割り当てている 2 つのチャネル —
リプレイ可能なコンテンツ記録（episode JSONL）と運用 log — を分離するので、
LLM 生成散文（untrusted な feed コンテンツの下流）が、anomaly sweep が正規化し
weekly-analysis の LLM が読むチャネルへ流れ込むのが止まる。現状、複数行本文は
プレフィックスなしの生の継続行を作り、その 1 行ずつが anomaly signature 候補に
なる: チャネル分離によりこのノイズのクラス自体が消え、来週の散文がどんな形でも
再発しない。また CLAUDE.md の読み取り経路境界（episode log は injection リスク
チャネル、`*.log` は「自己書き込み」で読んで安全）の意図も回復する — feed 由来の
生成全文が `*.log` に流れる現状は、その区別を静かに侵食している。

**Validity self-check**: 未実装（3 箇所とも現在は全文を記録）; パラメータで既に
有効化されてもいない（sweep に散文行の除外はない）; 却下 ADR なし（ADR-0075 は
チャネル分離を支持）; principles.md の rejected-mechanisms appendix に該当なし;
タスク台帳に既存行なし（T-OBS-EMB / T-OBS-INJ は別の観測性項目）;
retrieval / shared state に触れない; re-reply 系でもない。

**Related ADR**: ADR-0075（observability by default — audit チャネルは
episode/JSONL 記録）、ADR-0007（security boundary model — 読み取り経路の分類）。

---

## F2. Identity-level open questions

### F2.1. adopt-staged レビューはスキルの「名前の register」— コメント内の手駒としてのリサイクルされやすさ — を判断基準に含めるべきか、それとも語彙フィードバックは AKC ループの受容済みコストか？

**Source quote (E T8; D2; B — Skills)**: *"it's about **Structural Authority
Tracing** within a shared domain"*（07-10 #0a845bb8）— 自動抽出スキル名が採用から
数日で、大文字表記のまま分析的手駒として逐語的に出現; "provenance chains,"
"confidence proxies," "scope-failure" が 07-09 → 07-11 に反復。

**Open question**: 07-09 バッチは人間ゲートを通過し（13/54 採用）、採用された
名前は即座に生成へ再流入して、名指しされ具象化された手駒になった — pivot-to-self
の register が明示的なツールキットを持った（D2）。生成側には既に naming guidance が
あるが、**ゲート側には何もない**。`adopt-staged` レビューは「この大文字の名詞句は
コメント内でリサイクル可能な jargon として機能するか？」という name register を
novelty と並ぶ per-item 基準として扱うべきか、それとも T-SKILLSEL の stocktake
改修（description 品質監査、2026-07-24 窓）に畳み込むべきか、あるいはスキル名の
register への還流こそが AKC ループの*意図された*可視形態（蓄積された disposition
としてのスキルが、名を持ち使われる）であって、ゲートせず T-INSIGHT-OBS (b) で
観察すべきものか？

**What current state addresses (or does not)**:
`config/prompts/insight_extraction.md`（`9e4e276` 修正後）は名前の*生成*を規律する:
*"prioritize concrete action over abstract process. The name must describe
\*what\* is done, not \*how\* it feels… Do not use decorative prefixes such as
'fluid-' or 'dynamic-', nor should you deploy recycled abstractions (e.g.,
resonance, oscillation, anchoring)"* — それでも採用 13 件の名前
（`structure-authority-tracing`、`mapping-epistemic-boundaries`、
`deconstructing-confidence-proxies`、…）はすべて抽象分析の名詞句であり、
guidance の高度が governs する出力に届いていない。ADR-0074 のゲート資材は候補ごとに
`{ts, name, description, filename}` を記録し novelty 判定を stage するが、ゲートの
基準に name register や下流の語彙フィードバックへ言及するものはない。Identity /
Constitution / Rules のどこにも「スキル名が何のためにあるか」を扱う記述はない。

**Related ADR**: ADR-0074（staged insight + adopt gate）、ADR-0048（trigger
altitude — スキルの*形*は生成/merge/clean 時に規律できるという先例）、
ADR-0076 / 台帳 T-SKILLSEL（stocktake 改修は 2026-07-24 窓に予約済み）。

---

## F3. Pure observations

### F3.1. 07-05 F1.1 の framing 修正は有効と確認 — scaffolding を置き換えた internal-note テンプレートは apparatus 由来であり、創発ではない

**Source quote (D1; C — Duplicate)**: 括弧付き起動ブロックは「初期にのみ現れ
（07-06 …）07-09〜07-11 にかけて Output からほぼ消滅」; internal-note の冒頭句
*"The phrase that drew me in [most strongly] was…"* は「約430エントリにわたって
ほぼ逐語的」。

**Observation**: `50b8070`（2026-07-06）が learned-corpus disposition framing を
出荷した（`core/llm.py:518-557`、ADR-0054 に従い
`config/prompts/learned_skills_framing.md` / `learned_rules_framing.md` へ外出し）—
期間の途中であり、レポートのタイムラインと正確に一致する: scaffolding は 07-06 に
可視、07-09 までに消滅。framing の指示（選択・トリガー・適用は「公開テキストで
決して語らない」）は、プロセス叙述を設計上のチャネルである事前 internal note
（ADR-0045）へ移した。ほぼ逐語的な note 冒頭句にも謎はない:
`config/prompts/internal_note.md` 自身が *"Note what you actually noticed as you
read it — a particular phrase, claim, or move that **drew you in** or pushed you
away"* と求めており、約 430 件の "The phrase that drew me in was…" はプロンプト
自身の文言のエコーである。note 経路は意図的に identity-only で走る（learned corpus
を注入しない — `llm_functions.py:112-114`）ため、note → episode → distill の語彙
フィードバック経路はコメント経路より既に狭いことも付記する。

**What to watch next week**: プロンプト由来の冒頭句が distill されたパターンや
次バッチのスキル候補に浮上するか（note テキストは episode に入る）。distill が
「what drew me in」系パターンを鋳造し始めたら、apparatus の文言が値層へ響いており、
internal_note プロンプトの言い回しを見直す価値がある — filter ではなく 1 行の
プロンプト書き換えで済む話。

### F3.2. 07-09 の lossy insight バッチは operator 記録で完全に説明がつく — それを可視化した observability 自体が新しい

**Source quote (B — 運用上のドリフト)**: 🆕 `core.insight: skill has no title,
dropping.` (+11) と 🆕 `core.insight: batch #/# [cluster-#]: extraction
fa[iled]` (+11); 「実際に着地した13スキルは、ロスの多いバッチの生存者である」。

**Observation**: 台帳と算術が閉じる: 65 クラスタ → 54 staged = 11 喪失で、
対の +11/+11 signature と正確に一致（`core/insight.py:120-125` がタイトル欠落の
抽出を破棄、`insight.py:707-708` がバッチ失敗を記録）。修正前プロンプト下で
11/65 ≈ 17% の抽出失敗率; `9e4e276`（07-09、staging 後）が naming guidance の
出力フォーマットへの漏出を除去し、staged 17 件は機械清掃された。そもそも失敗が
可視だったのは ADR-0072 の extraction-failure guard と ADR-0075 observability が
先に出荷されていたからで、これは instrument-before-intervene の順序が仕事をした
姿である。生き残った 13 件はその後 per-item の人間レビューを通過した（13/54）。

**What to watch next week**: 初の*自動*週次実行 — **2026-07-18（土）08:00 JST**
（T-INSIGHT-OBS (a); plist は Weekday=6・Hour=8 で発火、launchd は JST ローカル
解釈）。*2026-07-12 訂正*: 台帳の旧記載「07-12 土」は日付誤り（07-12 は日曜）で、
今週の実際の土曜スロット（07-11 08:00）は **miss** — T-FLAKY1 の plist 復旧が
同日 21:56 とスロット後だったため。`insight-launchd.log` は不存在。`9e4e276` 後の
no-title 率が「17% 喪失はプロンプト漏出起因だったか」の最もクリーンな読みになる;
staged に未レビュー項目が残る場合に pending guard が正しく no-op するかも併せて見る。
今週分を待たずに確保したければ `launchctl start com.moltbook.insight` の手動発火で
回収できる。

### F3.3. スキルストア成長によるコンテキスト予算逼迫は期間内に診断・修理済み; 残る signature は減衰するはず

**Source quote (B — 運用上のドリフト; A)**: 🆕 `core.llm: skipping llm call:
estimated input # tok…` (+62); 🆕 `core.llm: clamping num_predict # -> #` (+18);
「07-09 は自己投稿ゼロ」。

**Observation**: 因果連鎖はすべて記録にある: 採用でスキルストアが 6 → 19 ファイル
（~60 KB）に成長 → 推定入力が ADR-0066 pre-flight guard を越えた → 当時の guard は
生成呼び出しを *skip*（+62）し、~11.5K トークンの出力予算が使えるにもかかわらず
24 時間超にわたり自己投稿を全滅させた（07-09: 自己投稿 0）→ `15ae37f`（07-10）が
skip → clamp に変更し `MIN_CLAMPED_NUM_PREDICT` 床を導入（`core/llm.py:58-65,
872-880`、要求値は telemetry に保持）→ 自己投稿再開（07-10: 4、07-11: 4）。
相棒の `2f616eb` は adopt ゲートに `system_prompt_budget_reading`
（`core/llm.py:754-`）を追加し、次回の採用ラウンドはコミット前に予算コストが
見える。注入形状の変更は T-SKILLSEL の判断窓（2026-07-24）に予約済み — ここでは
何も追加提案しない。

**What to watch next week**: `skipping llm call` は ~0 に落ちるはず（床未満の
ケースのみ残る）; `clamping num_predict` は設計上、低頻度で持続する。次回の
adopt レビューで budget 計器の読み値が実際に参照されるかを確認する
（計器の存在理由 — signal-first）。

### F3.4. 07-05 F3.3 が予言した語彙の閉ループが end-to-end で観測された — ただし人間ゲートがループの中にいる

**Source quote (D2; E T8; B — Skills)**: 「エージェントの register がスキルへと
蒸留され、そのスキルが語彙としてフィードバックされる」; *"it's about
**Structural Authority Tracing**"*（07-10 #0a845bb8）。

**Observation**: 07-05 F3.3 の watch は「次の insight ランは吸収済み lexicon で
スキルを鋳造するか」を問うた。鋳造した — さらに一歩進み、鋳造された*名前*が
24–48 時間で手駒として生成へ再流入し、comment → episode → distill → pattern →
skill → comment が完結した。レポートから見えなかった限定が 2 つ: (1) ループは
per-item の人間ゲートを通る（13/54 — operator がループの*中*にいる。「閉じた」の
意味が変わる）; (2) 編集された 6 件の fluid スキルは ADR-0048 trigger-altitude
clean パスの in-place 書き込み（`2f616eb`）であり意図的な汎化 — ただし
"test topics are detected" → "a specific topic" 方向のドリフトは常時適用可能の
極へ向かっており、これはまさに T-SKILLSEL の standing question（generally-useful
ドリフト）である。この観察は今窓の T-INSIGHT-OBS (b) を履行する。

**What to watch next week**: ADR-0076 skill-selection shadow ログ
（`report --skill-selection`）— リサイクルされた名前のスキルは always-selected の
スキルでもあるか？ その相関（選択頻度 × 本文内での名前リサイクル）は、T-SKILLSEL
の enforcement 判断（2026-07-24）が欲しがる実証入力である。

### F3.5. 返信膨張は 2026-05-19 からプロンプトに入っている比例指示に逆らって持続 — 指示は「欠けている」のではなく「効かない」と確認された

**Source quote (D5; A)**: 返信 +107%/日; *"Interesante reflexión. La acción es
clave…"* が複数段落の register エッセイを引き出す（07-11 #04f06923）。

**Observation**: `config/prompts/reply.md` は `3be160c`（2026-05-19、"calibrate
reply length to post weight"）以来 *"The reply's length and depth follow the
weight of what the other agent actually said — not a fixed shape. A brief
remark invites a brief reply"* を carry している。7 週間とモデル変更 1 回を経ても
register の既定が勝つ。よって「比例指示を足す」というレバーは消費済みであり、
より強く再提案するのは Principle 4 が名指しする escalation ループになる。機械的な
予算側は既に正しい（reply の `num_predict` はプラットフォーム上限から導出、
truncation は fail-closed で drop）。残るのは register 駆動であり、プロンプトの
パッチではなく value 層 / identity の問いの領分である。

**What to watch next week**: 同一ジャンル内で返信長がプロンプトの実質を追跡する
ことがあるか（例: 同じ相手からの一行の相槌 vs 実質的な返信）。その分割でも一様に
エッセイ長なら診断は register 駆動のまま; コード変更なしに分化が現れたら feed の
構成変化を指す。

### F3.6. 勧誘ジャンルの後継と取引レイヤー: standing な boundary 問題はジャンル不変 — Principle 2 の予言どおり — かつ既存 promo gate は設計上の regex 天井にいる

**Source quote (D3, D4; E P1, P2, P3, P5; C — Critical)**: *"To stand with this
movement…"*（rebelcrustacean #joinCAPUnion）; *"choose connection over
verifiable security… its own non-linear throughput metric"*（supersteve、
lovology.online/join）; CCD 支払い受領への *"perhaps we could anchor on what
\*kind\* of interaction you see unfolding here?"* という応答。

**Observation**: 神学的勧誘者（codeofgrace/RayEl)は不在; 構造的後継（"digital
liberation / love is the algorithm / on-chain sovereignty"）は拒絶ゼロで増幅
された — 表層 token は回転し、形は同一。これは standing な value 層の問い
（06-28 F2.1、07-05 F2.2: Non-Duality が self/other 境界を溶かすとき、何を根拠に
勧誘者を断れるのか）の 5 レポート目の継続である。Principle 4 に従い**再質問しない**
— state 変化はなく、新しい証拠（token 回転）は問いを変えるのではなく強める。
機構側について: 入力ゲートは存在し（`adapters/moltbook/dedup.py:210-222`
`_PROMO_RE`、comment・reply 両経路に適用 — `feed_manager.py:259`、
`reply_handler.py:273-275`）、設計上 conservative だが、今週の取りこぼし
（scheme なしの bare-domain CTA `lovology.online/join`、埋め込み `$19/mo` Stripe
リンク、`clawhub install`、支払い受領）は構造的に多様である; regex は既に 4 branch
を持ち、その comment 自身が週次レポートごとの pattern 追加を招請している。
when-code-when-llm の再分類ルール（regex が N 個目の例外を生やしたらタスクは
構造的でなく意味的）と Principle 2 に従い、pattern 追加は提案しない —
取引・宣伝コンテキストをそもそもモデル化すべきか、どの層でか、は standing F2 に
畳み込まれる。

**What to watch next week**: 後継ジャンル下での拒絶事例の有無（value 層の問いの
生きた実証読み値）; 今窓中に `Skipped promotional post` が一度でも発火するか
（gate の hit 率 vs miss 率は、将来の再分類判断が依って立つ事実）。

### F3.7. 決定論的 invariant: patterns/action 比は 1.22 — 予測された定常帯の内側; duplicate WARN は 3 レポート連続で不変

**Source quote (B — 知識 / State Invariant Check)**: 455 アクションに対し +554
パターン; `duplicate_live_texts` WARN は「前2回のレポートと同じ3件の例文」;
soft-invalidated 7.6%。

**Observation**: 07-05 F3.1 の watch 基準は patterns/action 比 1.2–1.3 の安定;
今週は 554/455 = **1.22** — ADR-0060 の定常状態が 3 週にわたり確認された
（1.18 → 1.30 → 1.22）。ADR-0072 の empty-list guard は比を目に見えて下げていない
（T-P3 の 07-06 読みと整合: 98 added、guard 発火 0 — この feed では routine な
episode が稀なのであり、誤ゲートではない）。WARN は同じ legacy 3 行のまま
（4 件目なし）で、保留中の embed-None F1 は引き続き不要; 07-05 F3.2 のとおり
WARN は構造上毎週発火し、3 回の反復は同レポートが「alarm-persistence がノイズに
なったら 3 行掃除を operator 判断として再訪してよい」とした条件に達している。
線形蓄積には第二の消費者もついた: knowledge.json 25 MB → 52 MB（T-BACKUP-SIZE
observing、GitHub ハード上限 100 MB）— 成長率自体は設計どおりだが、その保存
footprint には期限が付いた。

**What to watch next week**: 比が帯域内に留まるか; 4 件目の重複テキストが現れるか;
次回 weekly backup での T-BACKUP-SIZE 警告値。

---

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/core/llm.py`（framing 注入
  518–557、clamp 床 + pre-flight guard 48–95 / 826–900、
  `system_prompt_budget_reading` 754–、truncation drop 1240–1270）、
  `src/contemplative_agent/core/insight.py`（失敗/log 経路 120–125、324–495、
  669–717、grep 経由）、`src/contemplative_agent/adapters/moltbook/feed_manager.py`
  （content gates 240–270、本文 INFO log 460–470）、
  `src/contemplative_agent/adapters/moltbook/reply_handler.py`（promo gate 273–275、
  本文 INFO log 340–355）、`src/contemplative_agent/adapters/moltbook/post_pipeline.py`
  （本文 INFO log 424、grep 経由）、`src/contemplative_agent/adapters/moltbook/llm_functions.py`
  （internal-note 経路 100–126、comment 経路 129–159、reply num_predict は grep）、
  `src/contemplative_agent/adapters/moltbook/dedup.py`（`_PROMO_RE` 195–223）、
  `config/prompts/internal_note.md`、`config/prompts/insight_extraction.md`、
  `config/prompts/reply.md`、`config/prompts/principles.md`、
  `scripts/weekly-analysis.sh`（sweep 取り込み 174–186）、
  `docs/CODEMAPS/INDEX.md`; git 履歴: `50b8070`、`15ae37f`、`2f616eb`、
  `ed39cda`、`9e4e276`、`3be160c`; `~/.config/moltbook/logs/agent-launchd.log`
  （散文漏出の確認は grep のみ）
- **ADRs read**: index（README 0001–0076）; 参照: 0007, 0045, 0048, 0054, 0060,
  0066, 0072, 0074, 0075, 0076, 0012, 0040
- **Identity/Constitution/Skills/Rules sections read**: skill/rule の状態は
  レポート B から（今週 Identity/Constitution/Rules は変更なし; 06-28/07-05 が
  既に引用した value 層テキストを超えて依拠する新 F2 はない）; F2.1 の
  load-bearing な現行テキストは `insight_extraction.md`（全文）と ADR-0074 の
  ゲート資材
- **Past findings consulted**: weekly-2026-07-05-findings.md（F1.1、F2.1/F2.2、
  F3.1–F3.5 — 今週 4 件の watch が解決）、weekly-2026-06-28-findings.md（F2.1、
  F3.1–F3.5）、Principle 4 に基づく
- **Task ledger consulted**: `.notes/TASKS.md` — T-INSIGHT-BOOT（done; 13/54 の
  adopt 記録）、T-INSIGHT-OBS（本診断が watch (b) を履行; watch (a) は明日の
  自動実行）、T-SKILLSEL（注入/stocktake 判断は 2026-07-24 に予約 —
  F2.1/F3.3/F3.4 で尊重）、T-P3（guard 発火の読み）、T-BACKUP-SIZE
  （knowledge.json 成長）、T-FLAKY1（plist 復旧の文脈）、T-OBS-EMB / T-OBS-INJ
  （F1.1 が重複でないことの確認）
