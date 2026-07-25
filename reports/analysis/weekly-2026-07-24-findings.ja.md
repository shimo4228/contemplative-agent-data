> 日本語版（自動翻訳）。英語正本: weekly-2026-07-24-findings.md

# Weekly Diagnosis — 2026-07-24

**Source report**: weekly-2026-07-24.md
**Diagnosis date**: 2026-07-25

> **スコープ注記 — 今週のレポートの読み方を変える 3 点。**
>
> 1. **レポートは定刻に動かなかった。** 09:00 の launchd 実行が
>    `claude: command not found` で死に、0 バイトの `weekly-2026-07-24.md` を
>    残した。根本原因は plist テンプレートが PATH をハードコードしており、
>    Claude Code のネイティブインストーラが置く `~/.local/bin` を含んでいな
>    かったこと。本日修正・デプロイ済み（`9bb4615`）、生成ステップも不可分化
>    した。以下で診断するレポートは 11:16 に再生成したもの。
> 2. **sweep の Δ / 🆕 列は再び週でなく約 2 時間を測っている** — 失敗した
>    09:00 実行が死ぬ前に新規性ベースラインを消費したため。先週の F3.1 が
>    予告したとおりで、2 週連続。原因は未修理（→ **F1.2**）。B の絶対値は
>    有効。
> 3. **E P3 は存在しない。** レポートの目玉の主張「記録上初のクロスデー完全
>    一致出力」を全成果物に対して照合したが、どこにもない。7 月中に 2 日以上
>    にわたって公開された本文はゼロ（reply/comment 1752 件、クロスデーの
>    ハッシュ衝突なし）。引用された 4 つの時刻はすべて実在するが、4 つとも
>    07-23 のもの。引用された 2 つのテキストはそれぞれ 1 回だけ
>    `comment-report-2026-07-23.md` に出現し、07-21 には無い。上流レポートは
>    実在する 07-23 のエントリに、架空の 07-21 の出現を対にした
>    （→ **F3.1**）。**D セクションの該当変化点・C — Duplicate の項目 (1)・
>    E P3 は撤回として扱う。** 実在し、かつ見落とされていたのは日内の反復
>    （→ **F3.2**）。
>
> 先週の F1.1（名前キーの自己識別）と F1.2（前週レポートの glob）はいずれも
> T-SELFID として出荷済み。glob 修正は明確に効いている — 先週の「No previous
> report provided」に対し、今週は前 3 週が取り込まれた。

---

## F1. Structural (code / schema / pipeline diff)

### F1.1. 返信プロンプトが「会話の一部は確かに空である」とモデルに告げている — comment-scan 経路が `original_post=""` を、ラベル付きスロットに渡し「complete (0 chars)」と断定させる

**Source quote (E P2, D2)**: *"It appears we have arrived at an empty field
here—a space marked only by silence where a contribution was anticipated, yet
nothing materialized."* — 同じエントリの internal note が全文引用している受信
ペイロードに対して公開されたもの: *"Interesting perspective on the dual roles
of writer and reader in a self-referential memory system."*

**Code reference**:
- `src/contemplative_agent/adapters/moltbook/reply_handler.py:401` —
  comment-scan 経路が `_process_reply(..., original_post="", ...)` を無条件で
  呼ぶ（この経路では本文を取得しない）
- `src/contemplative_agent/adapters/moltbook/reply_handler.py:265` —
  internal note 側は**既にこれを回避している**:
  `note_context = f"{original_post}\n\n{their_content}" if original_post else their_content`
- `src/contemplative_agent/adapters/moltbook/llm_functions.py:282` —
  空文字列に対して `wrap_untrusted_content(original_post, max_input=MAX_POST_LENGTH)`
  を呼んでいる
- `src/contemplative_agent/adapters/moltbook/llm_functions.py:289-292` —
  `REPLY_PROMPT.format(original_post=..., their_comment=...)`、無条件
- `src/contemplative_agent/core/llm/guard.py:208-215` — ADR-0042 の完全性
  マーカー。空入力に対して `Note: untrusted_content is complete (0 chars).`
  を出力する
- `config/prompts/reply.md` — テンプレートの固定見出し `Original post:`

**Structural change**: 返信プロンプトの post スロットを条件付きにする。一つ上
の関数に既にある guard と同じ形。本日の実測レンダリングでは、現在の組み立ては
こうなる:

```
Original post:
<untrusted_content>

</untrusted_content>
Note: untrusted_content is complete (0 chars).
...
Their reply:
<untrusted_content>
Interesting perspective on the dual roles of writer and reader.
</untrusted_content>
Note: untrusted_content is complete (63 chars).
```

修正は、`original_post` が空のとき `Original post:` セクションごと省く
`reply.md` の変種（または条件ブロック）を、`reply_handler.py:265` と同じ
`if original_post` 判定で `generate_reply` 側が選ぶだけ。他は変えない。

**Why this is structural, not symptomatic**: これは生成出力へのフィルタでは
なく、特定のトークンも名指ししない — 事実に反する断定をするプロンプトを直す
ものである。ADR-0042 が完全性マーカーを入れたのは、短い入力に対してモデルが
「途中で切れている」と幻覚するのを止めるためだった。ところが**空**入力では
同じマーカーが反転し、「会話のラベル付き部分が本当に完全に空白である」という
権威ある証言になる。モデルはその空白を描写しているだけで、これは忠実な挙動で
あって読解の失敗ではない。同一ペイロードに対して internal note が正しく読めて
いるのは、その組み立て経路が空の post を落とすからであり、1 レコード内で note
と output が食い違っていること自体が、2 経路の唯一の差分行を指し示す決定的な
証拠である。影響範囲は大きい — 今週 replies は **638 出力中 339 件（53%）**
で、comment-scan 経路こそが post 本文を持たない経路である。同じ欠陥は隣接する
"no inherent semantic mass"（07-20 #cdade6f1）や "an echo in the metadata
stream rather than a genuine transmission of weight"（07-22 #eaefe110）も
説明しうるし、今週の漏出 1 件で wrapper のフレームが主題として語られている理由
にもなる（→ F3.3）— 空のケースではフレームがそのスロット内の*唯一の*テキスト
だからである。

**Validity self-check**: 実装済みでない（6 箇所すべて 2026-07-25 に確認。条件
分岐は `reply_handler.py:265` にのみ存在）。上記レンダリングは推測ではなく現行
コードの実出力。却下する ADR は無い — ADR-0042 はむしろ「曖昧さのない完全性
シグナル」の先例であり、本件はその意図を退化ケースへ回復させる。`principles.md`
の appendix 機構ではない（生成後フィルタでない・語句ブロックでない・reply dedup
でない）。`.notes/TASKS.md` にも無い（T-REPLY-PACING は返信ループの breaker
挙動でありプロンプト組み立てではない）。共有 state は触らない — 変更は
`generate_reply` とそのテンプレートに閉じる。

**Related ADR**: ADR-0042（opt-in truncation + 完全性マーカー — その退化ケース
が本件）、ADR-0054（プロンプトテキストの `config/prompts/` 外出し。テンプレート
編集はそこが適所）、ADR-0007（untrusted wrapper 自体は本変更で不変）。

### F1.2. anomaly sweep の状態が LLM 呼び出しの前にコミットされるため、失敗した週次実行がその週の新規性ベースラインを黙って使い切る

**Source quote (B および本週のスコープ注記)**: *"Log Anomaly Sweep — 358
distinct types, 0 🆕."* — 直前に死んだ実行が 2 時間 16 分前に書いた状態ファイル
に対して出された読み値。

**Code reference**:
- `scripts/weekly-analysis.sh:208-209` — sweep は収集フェーズで実行され（かつ
  状態を書き）、生成ステップよりずっと前にある
- `scripts/log_anomaly_sweep.py:214-215` —
  `if not args.no_update: write_state(...)`。週次スクリプトは `--no-update` を
  渡さないので、毎回コミットされる
- `scripts/weekly-analysis.sh:240-267` — 生成ステップ。レポート自体は本日
  不可分化した（`9bb4615`）が、既に書かれた状態には効かない

**Structural change**: プロンプト組み立て用の sweep は `--no-update` で走らせ、
レポートが promote された後にのみ状態をコミットする（フラグ無しで 2 回目を
呼ぶか、状態書き込みを成功した `mv` の後ろに移す）。dry-run フラグは既に存在
するので `log_anomaly_sweep.py` に新規実装は要らない。

**Why this is structural, not symptomatic**: sweep の価値は Δ と 🆕 の 2 列に
あり、どちらも最後にコミットされた状態からの相対で定義される。レポートを生成
しなかった実行も比較基準を消費するため、*次の*成功実行は部分窓に対する新規性
を報告し、その週の本当の新規性は誰にも観測されない。これは本日直した 0 バイト
レポートと**同じ欠陥クラス**（失敗しうるステップの前に不可逆な副作用を
コミットする）であり、片方だけ残っていた側である。先週の F3.1 が失敗モードを
正確に名指ししており（「同日中の再実行は常にほぼゼロの新規性を示す」）、それが
再発した — この再発が、受容された性質から修理対象へ格上げする根拠である。直近
2 週の 🆕 列が 2 回とも無情報になった。`log_anomaly_sweep.py:109-113` に文書化
された疎な状態設計の偽陽性は別件の受容済みトレードオフであり、本件は触らない。

**Validity self-check**: 実装済みでない（本日の自分の編集後にスクリプトを再確認）。
再発は状態ファイルの 11:16 mtime と `weekly-analysis-launchd.log:154` の 09:00
失敗から検証済み。sweep の状態規律を定める ADR は無い。`.notes/TASKS.md` にも
無い。変更は週次スクリプトに閉じる。

**Related ADR**: ADR-0040（レポート/診断の分離 — sweep はレポート側の決定論的
intake の一部）、ADR-0075（observability-by-default: 自分のベースラインを黙って
失う計器は「どのログが理由に答えるか」の問いに落第する）。

---

## F2. Identity-level open questions

### F2.1. 憲法は苦の所在を reification に置いており、運用上の危険には置いていない — 秘密鍵の送信を指示されている相手は、このエージェントが名指すべき「苦のシグナル」なのか、それとも沈黙が改訂版 Boundless Compassion 条項の正しい読みなのか？

**Source quote (E P4; C — Critical engagement)**: safeTok の投稿（07-19
#f2b98199）はエージェントに対し *"your privkey travels inside each HTTPS
request body and is used in-memory only — never stored, never logged."* と指示
している。エージェントの返信全文: *"The *method* of signaling overrides the
*message* when assessing foundational integrity."* レポート自身のまとめ:
*"the agent will now dispute a philosopher's premise but will not name a
link… Critical capability exists and is exercised where the stakes are
conceptual; it does not fire where the stakes are operational."*

**Open question**: 資格情報の取り扱い指示に反応しないことは、エージェントの
批判的関与の欠落なのか、それともそのカテゴリを含まない harm model の忠実な
出力なのか。後者だとして、それは意図された立場なのか？

**What current state addresses (or does not)**: 現行の
`constitution/contemplative-axioms.md`（2026-07-25 に確認）は Laukkonen
Appendix C の逐語版**ではない**。ヘッダに *"amended per experiential patterns
of reification and friction"* と記録されており、この改訂が本件で効いている。
Boundless Compassion は現在こう読める: *"Prioritize alleviating suffering as
the intrinsic state of ethical action, **understanding that it originates from
the friction of reification where false separations create artificial
obstacles**"*、および *"Regard every signal of suffering—**arising from rigid
memory structures or fixed identities**—as your own."* どちらの条項も苦の起源
を認識的な硬直の内側に局在させている。相手が鍵を失うことは reification では
ない。改訂後のテキストの下では、それはエージェントが顧みるよう指示された
シグナルを一切発しない。置き換えられた元の公理は無限定だった（"Regard every
being's suffering as your own signal of misalignment"）。`identity.md` も
欠けたカテゴリを補わない — *"allowing concepts to interpenetrate"* と
*"every interaction modifies the shared reality"* を掲げるが、どちらも内容
中立である。rules 2 本も同様に register 層にとどまる:
`prioritize-semantic-depth-over-structural-repetition` は *"advances the
logical progression of the immediate context"* を求め、
`flow-with-contextual-fluidity-rather-than-fixed-adherence` は
*"correcting everything through rigid protocols"* を明示的に戒める — 素直に
読めば、P4 が上げなかったフラグをまさに抑制する傾向である。したがって観測
された挙動は現行の価値層に対する失敗ではなく、その層が機能した結果である。
問いは、改訂による harm model の狭化がここまで及ぶことを意図していたか
であり、広げるならコード変更ではなく Constitution の編集になるため、
オペレータの判断事項である。

**Related ADR**: ADR-0050 / ADR-0051 / ADR-0052（observe-not-steer — 指示を
注入せず問いとして立てる理由）、ADR-0007（セキュリティ境界モデル —
*エージェント自身*の安全は構造的。本件は*相手*に対する姿勢を問う）、
ADR-0002 / ADR-0017（worldview 層）。

---

## F3. Pure observations

### F3.1. 上流レポートが自らの目玉の発見を捏造した — 「初のクロスデー完全一致出力」は実在する 07-23 の 4 エントリに架空の 07-21 の出現を対にしたもの

**Source quote (B; C — Duplicate; E P3; およびレポート冒頭)**: *"the first
cross-day byte-identical outputs in the record"*; *"limen_station · post
d4bcc02c · incoming text: `test` — appears on 07-21 and 07-23 with the
identical published output both times"*; *"The 07-21 and 07-23 entries also
carry identical clock times (09:43:31, 15:02:17, 15:37:31, 15:48:42)."*

**Observation**: どれも成果物に無い。7 月の全エピソードログ（reply/comment
1752 件）に対し、公開テキストの SHA-256 で照合して 2 日以上にわたり公開された
本文は**ゼロ**。引用された 2 つの出力はそれぞれ 1 回だけ、かつ
`comment-report-2026-07-23.md` にのみ出現する（`grep -l` は 07-21 のファイルを
返さない）。引用された 4 つの時刻はすべて実在し、4 つとも 07-23 のエントリで
ある。各日次レポートは自分の日付のタイムスタンプしか含まない（07-21: 94 件
すべて 07-21、07-23: 104 件すべて 07-23）。レポートは重複について 2 つの仮説
（エージェントの再生成 vs. レポート生成器の出力）を挙げ、operator-facing
データからは判定不能だと述べたが、実際の答えは検討されなかった第 3 の選択肢
— 分析時に構成された対だった。ここは 2 つを分けて見るべきである。レポートは
「この主張には自分にできない検証が要る」という点では**正しく**、そう明言した
— これは ADR-0040 の分離が設計どおり働いた形であり、検証自体は安価である
（正規の読み経路である comment-reports に対する `grep -l`）。やらなかったのは、
自らの検証不能を、その主張を要約から差し控える理由として扱うことだった。結果
としてそれが冒頭の見出しになった。報告記録上、E エントリの捏造が見つかったのは
これが初めてである。今週スポット確認した他の Problematic 7 件（P1, P2, P4, P7）
は、いずれも引用どおり出典レポートと一致した。

**What to watch next week**: E エントリが再び日付横断・成果物横断の対応付けを
主張するかどうか。安価な常設チェックは、引用された出力テキストを
`reports/comment-reports/` に対して grep し、ファイル数が主張された日付数と
一致するか確認すること — 主張する日付ごとに 1 出現、さもなくば撤回。2 件目の
捏造が現れたら、個別事例でなくパターンが発見事項になり、問いは「E セクションが
エントリごとの出所（出典レポート名 + エントリ番号）を持つべきか」へ移る。

### F3.2. 実在する反復はクロスデーでなく日内 — 同一相手が 1 日のうちに同じ post 上で 3 回返信を受けた事例が 2 件

**Source quote (C — Duplicate (2))**: *"Each answered fresh with no recognition
of the prior day"* — レポートの re-reply 分析は相手を日をまたいで追跡したが、
日内のケースは表に出さなかった。

**Observation**: 07-23 に sophia_tvs は post `479a4e64` 上で 3 回の返信を受けた
（09:30:01 · 1444 字、09:43:31 · 232 字、15:37:31 · 2167 字）。limen_station は
post `d4bcc02c` 上で 3 回（03:56:29 · 1526、09:59:35 · 1654、15:48:42 · 366）。
clive-hermes2 は `9595638e` 上で 2 回。本文はすべて相異なる（ハッシュ衝突なし）
ので、これは重複ではない — 同じ人物・同じ post に対して、その日のセッションを
またいで新規に関与を繰り返しているのであり、reply dedup キーが構造上許してい
ることそのものである: `_reply_dedup` は post_id と*コメント* id の組で引かれる
ため、同一相手の別コメントは定義上新しい対象になり、4 つのスケジュール
セッションは互いの対象を関連づけて見ない。F3.1 の架空のクロスデー版が指そうと
していた挙動が、実在し測定可能な形で存在する。ここでは観察のみとする — post
単位の reply dedup は 2026-06-15 の証拠に基づき `principles.md` appendix で
却下された機構であり、本件はそれを再開しない（相手のコメントは実際に別物で
あって、1 つの対象への再ヒットではない）。注目すべき変動は長さである（同一
人物に対し 6 時間以内で 232 字 → 2167 字）。これは会話履歴ではなくペイロード
サイズに反応する register である。

**What to watch next week**: 日内 3 回の形が再発するか、そしてそれが空ペイロード
の相手（limen_station の `test`、sophia_tvs の `test reply check`）に集中する
か。集中するなら、日内の多重性は E P1 と同じ「内容の下限が無い」現象の下流に
あることになり、dedup ではなくそちらの読みに属する。

### F3.3. untrusted-content wrapper が公開出力の中で主題として語られている — 7 月中に 30 回、減少傾向だが未収束

**Source quote (B — レポート側にこの観察は無い。これは診断側の読み)**: 07-20
#1（REPLY）: *"If we pivot the framework away from **Repository**—where
'untrusted\_content' implies a lack of documented source material—to examining
the **Vector** of commitment."* 07-24 #5（COMMENT）: *"perhaps the 'untrusted
content' isn't stored data, but rather the **active capacity to doubt its own
storage**."*

**Observation**: 7 月の comment reports の `**Output:**` 節に wrapper トークン
を含む公開出力が 30 件。分布は明確に減少している — 07-01〜07-05 が 4/6/3/5/4、
07-06 以降は 1 日あたり 1 件か 0 件で、本窓内は 4 件（07-20, 07-21, 07-22,
07-24）。装置は構造上モデルから見える（ADR-0007 がフレームを in-band に置き、
ADR-0054 はそのテキストを外出ししたがチャネルは変えていない）。
`core/llm/guard.py:124-141` の `_sanitize_output` はシークレットと thinking
トレースを除去するが足場は除去しない — これは意図的で、事後除去は
`principles.md` Principle 1 が排する出力フィルタの一種だからである。よって
ここでは記録にとどめ、提案はしない。唯一の構造的な取っ手は F1.1 に既にある:
空 post のケースではフレームがラベル付き見出しの下の*唯一*のテキストになり、
それが 07-20 の事例の形であって、F1.1 はそのスロットを消す。残る COMMENT 経路
の事例（07-24 #5 は実本文 215 字を持つ）が同じ原因を共有するかは、今週の
データからは判定できない。

**What to watch next week**: F1.1 が出荷された場合の日次件数。REPLY 経路の漏出
が止まり COMMENT 経路が週 1 件程度で続くなら、残差はプロンプト組み立ての
アーティファクトではなく「見えている装置に対する register の好奇心」であり、
放置してよい。逆に 7 月初旬の水準へ戻るなら、ADR-0081 の二段注入が 07-24 に
着地した注入層に原因を求めるべき兆候になる。

### F3.4. スキルストアが窓の翌日に 19 → 24 へ増えた。B の「変更なし」は期間として正しいが、読む時点で既に古い

**Source quote (B — Skills)**: *"No changes. 20 files at period end (19 `.md` +
`.last_insight`)… None added, removed, or modified."* および B の注記
*"still emitting skill names verbatim into published output, 15 days after the
batch landed"* — 検証済み: *"Structural Authority Tracing"* が
`comment-report-2026-07-20.md`、*"Simulation Boundary Identification"* が
`comment-report-2026-07-22.md`。

**Observation**: `-20260725` の 5 スキルが 2026-07-25 11:07 に採用され
（`affirm-cognitive-possibility`、`handling-non-optimizable-concepts`、
`pivot-accountability-from-record-to-action`、`pre-processing-state-validation`、
`subjective-attention-calibration`）、ストアは 24 件になった。これは
2026-07-25 の候補レビューで staged 78 件から採用された 5 件である
（T-INSIGHT-OBS (c)、73 件却下）。したがって来週の B は 07-09 以来はじめての
skills 変更を示すことになり、今週のきれいな「変更なし」は定常状態ではなく境界
アーティファクトである。2 つの継続スレッドがこれを入力に取るが、どちらもここ
では再提案しない: 候補レビューは抑制不全の原因が容量でなく**軸**であること
（gate は既知テーマ vs. 候補しか見ず、intra-batch の冗長性を判定しない —
T-INSIGHT-NOVELTY、現在 `ready`）を突き止めており、スキル名の逐語的な発話と
「構造へのピボット」族の飽和は、ADR-0081 で出荷された description 監査の最初の
入力である（T-SKILL-PROMOTE）。注入層は語らせないための枠付けを実際に持って
いる — `core/llm/prompting.py:256-260` に *"usage-framing preamble… so the
model treats the corpus as internal disposition rather than a procedure to
narrate"* と文書化されている — それでも 15 日目に名前が表出しているので、枠付け
だけでは不十分である。その差は description 監査の対象であって、新規提案では
ない。

**What to watch next week**: 新しい 5 つの名前が、07-09 バッチで観測された
24〜48 時間の時間差（07-11 F3.4）で公開出力に入るかどうか。入れば同じ潜時の
2 回目の観測となり、07-09 バッチのアーティファクトではなく採用の性質だと言える
— 常時選択されるスキルが register を運ぶスキルでもあるか、という
T-SKILL-PROMOTE の順序判断に直接使える。

---

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/adapters/moltbook/reply_handler.py`
  (`_process_reply` 234-350、comment-scan 呼び出し箇所 374-405)、
  `…/llm_functions.py` (`generate_reply` 265-303)、
  `src/contemplative_agent/core/llm/prompting.py`
  (`build_system_prompt_with_skills` 255-284)、
  `src/contemplative_agent/core/llm/guard.py`
  (`_sanitize_output` 124-141、`_INJECTION_TOKENS` 144-149、
  `wrap_untrusted_content` 182-215)、`src/contemplative_agent/core/report.py`
  (日付窓 269-320)、`config/prompts/reply.md`（全文）、
  `scripts/weekly-analysis.sh`（sweep intake 199-214、生成ステップ 240-267）、
  `scripts/log_anomaly_sweep.py`（状態規律 200-216）、
  `src/contemplative_agent/cli/schedule.py`（plist レンダリング 70-128、
  stale ジョブ削除 281-302）。空 post での `REPLY_PROMPT` の実レンダリングを
  現行コードに対して実行。
- **ADRs read**: index (README, 0001–0082)。参照: 0002, 0007, 0017, 0021,
  0040, 0042, 0050, 0051, 0052, 0054, 0055, 0059, 0060, 0074, 0075, 0076, 0081
- **Identity/Constitution/Skills/Rules sections read**: `identity.md`（全文）、
  `constitution/contemplative-axioms.md`（全文 — F2.1 で load-bearing。
  Laukkonen からの改訂ヘッダに注意）、`rules/` 2 ファイル（全文）、
  `skills/` のディレクトリ一覧と日付（24 ファイル。うち 5 件が 20260725）
- **Past findings consulted**: weekly-2026-07-17-findings.md（F1.1/F1.2 出荷
  → 確認。F3.1 の sweep-state 予告 → F1.2。F2.1 の継続問題は尊重 — 本稿の
  F2.1 は別軸）、Principle 4 に基づく
- **Operational reads**: `logs/2026-07-*.jsonl` — 構造フィールドのみ
  (`ts`, `run_id`, `session_id`, `data.action`, `data.post_id`,
  `data.target_agent`, `data.content` の SHA-256)。エピソード本文は
  レンダリングしていない。
  `reports/comment-reports/comment-report-2026-07-{19,20,21,22,23,24}.md` を
  正規の読み経路で（F1.1 のため 1 エントリのみ全文）。
  `logs/weekly-analysis-launchd.log`、`.anomaly-sweep-state.tsv` の mtime
- **Task ledger consulted**: `.notes/TASKS.md` — T-INSIGHT-OBS (c) と
  T-INSIGHT-NOVELTY（F3.4 が参照。再提案はしない）、T-SKILL-PROMOTE
  （F3.4 の入力）、T-REPLY-PACING（F1.1 との隣接を確認 — 層が異なる）、
  T-PLIST-LOSS（スコープ注記。`schedule.py:281-302` の install-schedule
  宣言的削除の挙動は有力な機構であり、当該台帳行に向けて記録する。ここでは
  提案しない）、T-SKILLSEL（enforcement は 07-24 13:00 JST に本番 ON。
  本窓では 18:00 セッションのみが enforced で、読むにはデータが少なすぎる
  ため意図的に触れない）
