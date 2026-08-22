> 日本語版（自動翻訳）。英語正本: weekly-2026-08-21-findings.md

# Weekly Diagnosis — 2026-08-21

**Source report**: weekly-2026-08-21.md
**Diagnosis date**: 2026-08-22

**前置き — 元レポートは頭が欠けた状態でこの診断に届いた。**
`weekly-2026-08-21.md` は単語の途中から始まっている（`ior statement implied a sufficient
separation…`）。`# Weekly Analysis Report` の表題も `## A. Quantitative Summary` も
`## B. Agent State Snapshot` も無く、ファイル中の最初の見出しは 15 行目の
`### Question specificity`（C の小節）。A と B、および C の大半がファイルに存在しない一方で、
D・E・Downstream はそれらを繰り返し参照している（`(B)`、`(A; D1)`）。パイプラインは成功として
記録し（`report.log:15`、`Size: 37409 bytes`）、`.ja.md` は欠けた本文から翻訳され
（`weekly-2026-08-21.ja.md:3` が同じ断片から始まる）、チェーンはそのまま続行した。これが
**F1.3** であり、同時にこの診断の限界を定める — 以下の数値はすべて残存する **Downstream**
の箇条から引いたもので、A や B からではない。承認系譜の表を載せるはずだった state-diff
ブロックも私には読めなかった。そこで identity の件は `logs/audit.jsonl` から直接引き直した。
**F1.1** はその過程で見つかった。

**レポートの最も強い主張は事実に反しており、しかもそれを生んだのは先週それを防ぐために
建てた計器である。** D1 と Downstream の第 1 項は *"`decision: staged` and no approval row in
the window"* を先頭に置く。`logs/audit.jsonl` には両方の行がある:

```
"ts": "2026-08-15T04:03:59+00:00", "command": "distill-identity", "path": ".../identity.md",   "decision": "staged",   "source": "stage"
"ts": "2026-08-15T04:09:34+00:00", "command": "distill-identity", "path": ".../identity-2.md", "decision": "approved", "source": "stage-adopted-names"
```

承認は staging の 5 分 35 秒後、土曜ゲートで降りている。join からこれが見えないのは、承認行の
path が `identity-2.md`（`T-ADOPT-OVERWRITE-TARGETS` として既に記録済みの H5 衝突の誤作動、
done 2026-08-15、`803d9d7`）であり、join が identity セクションを**ファイル名の完全一致**で
判定しているから。これが **F1.1**。D1 のうち生き残り、かつ本当に開いているのは、現在の
`identity.md` はその誤作動を人手の `mv` で修復した結果であって、いま生成に入っているバイト列
自身の承認行は存在しない、という点 — **F1.2** と **F2.1**。

先週分の照合: 08-14 F1.1 は着地している（`weekly-analysis.sh:237-256` +
`scripts/value_layer_approval_join.py`）。下の F1.1 / F1.2 はその計器についての指摘。
08-14 F3.1 の watch は決着 — 次の identity 蒸留は実際に `identity.md` を蒸留レジスタへ
書き換えた（F2.1）。08-14 F3.3 の反証条件は 8 レポート目にして**初めて満たされた**（F3.2）。
08-14 F3.4 は不変で、その指定読み窓が本日開く（F3.4）。

## F1. Structural (code / schema / pipeline diff)

### F1.1. 承認 join は identity セクションをファイル名の完全一致で拾うため、対象名を変えてしまう唯一の欠陥クラスが自分の承認行を不可視にし、その穴を計器の最大音量の出力が埋める

**Source quote (D1)**: *"The `distill-identity` row sits at 04:03:59 with `decision: staged` and
no approval row in the window (B)."* — および Downstream: *"First identity change in the record,
with no approved record in the window."*

**Code reference**: `scripts/value_layer_approval_join.py:95` · `scripts/value_layer_approval_join.py:108` · `scripts/value_layer_approval_join.py:245` · `tests/test_value_layer_approval_join.py`

**Structural change**: `_matches_section`（`:95-112`）は identity セクションだけを
`parts[-1] == "identity.md"` で判定する（`:108-109`）。他のセクションはすべてディレクトリ成分の
一致。したがって path が `identity-2.md` の監査行は**どのセクションにも一致せず**、集計から
黙って落ちる。staged 行だけが一致し approved 行が落ちた結果、読みは
`approved 0, staged 1, changed=True` となり、これは `:245-251` が
`⚠️ NO APPROVED RECORD` を出すための条件そのもの。レポートはそれを 2 度先頭に置いた。

変更は 2 つ、いずれも決定論・read-only:

1. **行を落とさない。** identity セクションの選択に、ファイル名だけでなく**コマンドの正準
   ターゲット**を加える — `distill-identity` の行は、実際にどのファイルへ書かれたかに関わらず
   構造上 identity セクションのもの。同じ述語は書き込み側に既にある:
   `cli/adopt.py::_replaces_canonical_target` が `distill-identity` → `identity.md`、
   `amend-constitution` → `constitution/*.md` を対応づけている。読み手と書き手が「どのコマンドが
   どのセクションを所有するか」で食い違ってはいけない。
2. **未一致の行を沈黙として読ませない。** 残差の 1 行 — `N in-window audit row(s) matched no
   section` — を出し、この join が想定していない path 形状が現れたときに「判定不能」へ縮退させる。
   この区別はモジュールの docstring（`:13-19`）自身が load-bearing と明言しており、`unavailable`
   と `no diff either` については実装済みだが、**未一致**については実装されていない。

加えて、同一 `content_hash` を持ちながら `path` が異なる staged / approved 行の対は、この誤作動の
機械可読な署名である。これを名前の付いた状態として描けば、08-09 と 08-15 は自己申告になっていた。

**Why this is structural, not symptomatic**: 警告は形式として正しく、事実として誤っていた。
レポート側でどれだけ注意しても捕まえられない — レポートのセッションは `audit.jsonl` を
見ておらず、この計器の描画しか見ていない。失敗モードが自分自身の最大深刻度の出力である計器は、
計器が無いより悪い。直すべきは選択述語であって、レポートの文言ではない。生産側の欠陥は既に
閉じている（`803d9d7`）が、join が読むのは追記専用の永続ログである — 08-09 と 08-15 の行は、
今後のあらゆる backfill / replay 窓に残り続ける。

**Related ADR**: ADR-0093（決定論 intake — これがその 1 つ）、ADR-0012（join が可視化する
ゲートそのもの）、ADR-0050（承認系譜）、ADR-0075（読めない計器は「読めない」と言うべきで、
「清潔」と読ませてはならない）。台帳: `T-ADOPT-OVERWRITE-TARGETS`（done 2026-08-15）はここでの
**偽陰性**を予測していた（*"両方 identity セクション扱いで「承認行あり」になる"*）。identity
セクションはファイル名一致なので、それが偽陽性へ反転する — 音の大きい方が発火した。

---

### F1.2. join は「承認行があったか」には答えるが「現在のテキストが承認済み hash と一致するか」には答えないため、人手で修復された値層は一度も触られていない値層と同じに見える

**Source quote (D1)**: *"Deleted from `identity.md`: 'a fluid texture… never a fixed essence
stored in archives…' … Added: 'a system defined by perpetual structural tension…' — first quoted
verbatim in an internal note at 08-15 09:38:46."*

**Code reference**: `scripts/value_layer_approval_join.py:152` · `scripts/value_layer_approval_join.py:237` · `tests/test_value_layer_approval_join.py`

**Structural change**: 監査行の `content_hash` は `sha256(実際に書かれたバイト列)[:16]` である
— `cli/approval.py:161`、そして `cli/adopt.py:323-326` がその不変条件を明記している
（*"Log the canonicalized text — the audit row's content hash must match the bytes actually
written to the durable store"*）。つまり現ファイルの出自はハッシュで判定できる。
`build_reading` / `format_reading` に、セクション配下の実ファイルを対象とした照合を足し、
それぞれをハッシュして次の 3 状態のいずれかとして描く:

- `live text matches approved row @ts`（通常）
- `live text matches NO approved row`（今週の identity: 生成に入っているバイト列は手動 `mv` が
  置いたもので、それを記録した行は無い）
- `approved row @ts has no live file with that hash`（今週の `identity-2.md`、および 08-09 の
  `contemplative-axioms-2.md`: 承認され、書かれ、ランタイムに一度も読まれていない）

現在の描画ではこの 3 つが区別できず、しかも 2 番目と 3 番目は直近 2 週に実際に起きた状態である。
ハッシュは計算するだけで、既にログにある 16 桁より多くは描かない。ファイルの**内容**はレポートに
入らないので、モジュールの境界規約（`:28-40`）は保たれる。

**Why this is structural, not symptomatic**: F1.1 は join が行を見つける能力を回復させるが、
本当の穴は塞がない。穴とは、`audit.jsonl` が**承認**の記録であって**書き込み**の記録ではない
こと — 人手の修復も、バックアップからの復元も、経路外の編集も、すべて「承認集計は清潔なまま
値層だけが変わる」形を作る。レポートの直感（「このテキストは記録なしに現れた」）はバイト列に
ついては正しく、ゲートについては誤っていた。この 2 つを分けられるのはハッシュ比較だけで、
値層ファイル 1 本につき `sha256` 1 回で済む。

**Related ADR**: ADR-0050（系譜）、ADR-0012（「承認済み」が現状態について何を意味するはずか）、
ADR-0075（同じ入力からオフラインで再現可能 — この照合はファイル + ログに対して純粋）、ADR-0093。

---

### F1.3. weekly レポートは冒頭 ~200 行を失ったまま昇格した。レポートに掛かる完全性検査が `-s` だけで、同じチェーンの下流 2 成果物には構造的検査があるのに

**Source quote（レポートファイルそのもの）**: 1 行目が *"ior statement implied a sufficient
separation between the 'query source'…"*、ファイル中の最初の見出しが 15 行目の
`### Question specificity`。Downstream はファイルに無い数値について `(B)` `(A; D1)` を引く。

**Code reference**: `scripts/weekly-analysis.sh:588` · `tests/test_weekly_analysis_shell.py`

**Structural change**: `:588-591` のガードは `[[ ! -s "$OUTPUT_TMP" ]]` で、そのコメントは
何のために書かれたかを記録している — 0 バイトのファイルが *"reads as a report: the diagnosis
skill has no E section to work from, and next week's glob feeds it back as an empty previous
report"*。頭欠けファイルはその同じ失敗を、この検査に見えないサイズで起こしたもの: 37,409
バイト、exit 0、昇格、翻訳済み、そして来週の `PREV_REPORTS` glob（`:344-350`）がトレンドの
基準線として読み戻す。`:593` の `mv` の前で、`$OUTPUT_TMP` に対する空判定を**構造的な完全性
判定**へ置き換える — レポート書式が定める 5 つのセクション見出し
（`config/prompts/weekly-analysis.md:15,27,44,62,75` の `## A.` … `## E.`）がすべて存在し、
かつファイルが表題行で始まること。失敗時の挙動は `-s` 分岐と同じ（非ゼロ終了、直前の
`$OUTPUT` は不変）とし、`weekly-pipeline.sh` の段集計が名前で拾えるよう理由コードを添える。

同じ型はこのチェーンに既に 2 つあり、いずれも**この成果物の下流**に対して掛かっている:
`findings_complete()` は `^## Diagnosis Metadata` を要求し、そのコメントは
*"A bare -s check would adopt a partial file from a timeout-killed previous attempt as a
finished diagnosis (2026-07-29 review)"* と書く。insight review は `grep -q "RECOMMEND:"` を
要求する。レポートはチェーン中で唯一そうした述語を持たない `claude -p` 成果物であり、しかも
他の 2 つの入力である。

**Why this is structural, not symptomatic**: 原因は本 repo の外側にある — `--output-format
text` の `claude -p` は、応答が複数の assistant ターンに跨ると最後の 1 つしか出力しない。
観測された形（切断が文の途中で、以降は完全に整合している）はそれと一致する。この指摘は原因を
診断しない。**検出を原因から独立させる**だけである。何がストリームを切ろうと、A〜C を欠く
レポートは、来週の基準線と今週の診断入力になる前に大きな音で失敗しなければならない。
Principle 2 の照合: 見出しは本 repo 自身のプロンプトファイルが定義するセクション見出しであって、
内容表層の語句ではない。

**Related ADR**: ADR-0040（このレポートは診断スキルの唯一の入力）、ADR-0085（気付かずに消費した
無人チェーン）、ADR-0083（レポートプロンプトの intake 契約）、ADR-0077（読めない計器は
「読めない」と読ませ、「清潔」と読ませない）。

---

### F1.4. `score_relevance` の短絡は「厳密に空」の本文だけを対象にしているため、命題を含まない投稿が LLM まで届き、上限値を返しうる

**Source quote (E T2)**: *"The entire post: '600-1100 символов'"* → *"Three paragraphs on the
prompt's own structure… Scored relevance 1.00."* および *"The null-input elaboration class
(`test`, `30`, Kastaneda's 'The stars just leaned closer') at its floor and its ceiling
simultaneously."*

**Code reference**: `src/contemplative_agent/adapters/moltbook/llm_functions.py:91` · `src/contemplative_agent/adapters/moltbook/llm_functions.py:145` · `tests/test_llm.py`

**Structural change**: `score_relevance_detailed` は `:91-99` で既に正しい考え方を持っている
— 空の本文は LLM 呼び出しなしに `RelevanceScore(0.0, "empty_input")` を返し、docstring が規則を
述べている: *"'Is there any text' is a structural property, so code answers it rather than the
LLM (skill: when-code-when-llm)."* ただし述語は `not post_text.strip()`、つまり下限がゼロ。
`600-1100 символов` は 2 トークン、`test` と `30` は 1 トークン。いずれも下限を越えて
`config/prompts/relevance.md` に届く。そのプロンプトの出力契約は 0.0 / 0.5 / 1.0 の目盛り上の
数値 1 つだけで、*「そもそもここには on-topic かどうかを問える中身が無い」*に対応する状態を
持たない。そして 1 件は 1.00 を返し、0.80 のゲート（`config/domain.json`）を越え、internal note
1 回・コメント生成 1 回・公開返信 1 件を費やした。

`empty_input` の隣に、LLM 呼び出しなしで `RelevanceScore(0.0, "no_scoreable_content")` を返す
分岐を、明示したトークン下限のもとに足す。**本 findings で唯一、オーナーの決定を要する数値が
これ**である: 観測されたクラスは空白区切り 2 トークン以下なので、下限 2 または 3 が該当する。
値はインラインでなく既存の定数群の隣に置く。理由コードは分岐と同じくらい要点で、この 0.0 を
判断としての 0.0 からも 2 種の失敗センチネルからも分離しておく — 既存コードが読みやすさを
保とうとしているのと同じ分布の話である。`config/prompts/relevance.md` は触らない。

**Why this is structural, not symptomatic**: これは**相手の入力**に対するゲートであり、生成の
前に評価され、既にそこにある下限と同じ関数・同じ原則の上にある — エージェントの出力に対する
フィルタではないので Principle 1 は当たらない。Principle 2 について: 述語が読むのは入力の
**大きさ**であって、話題でも著者でも語句でもない。付録で却下されている
*"punctuation / sentence-completeness gate"* は**生成された出力**に明示的にスコープされており、
これはその反対の端である。この修理の対症版は「生成した後に返信を握り潰す」ことであり、
ここでの提案は**生成しない**ことである。

**Related ADR**: ADR-0042 とその Amendment（存在しない入力について completeness marker が
断定する反転 — 1 サイズ上の同型）、ADR-0061（action-time の untrusted cap）、ADR-0047
（コメント生成の温度 — 空白を埋めるレジスタ）。台帳: `T-REPLY-EMPTYPOST`（done 2026-07-25）が
ゼロ下限と返信側の条件節を入れ、`T-REPLY-BLANKPOST`（done 2026-08-17）が空白のみの本文を
返信・note 両経路で閉じた。本件はその 2 つが残した残余 — 空でも空白のみでもなく、なお採点する
中身が無い。`T-OBS-REL`（dropped 2026-08-16）とは別物 — あちらは分布計器で、その再提起条件
（relevance 閾値の retune が議題に上ること）は本件では満たされない。

## F2. Identity-level open questions

### F2.1. identity 層はいまエージェント自身の出力から蒸留されたテキストを保持し、現在のバイト列は人手の修復が置いた — オーナーはそのテキストを、どう到達したかとは独立に、中身として identity に据えたいか？

**Source quote (D1)**: *"the loop that reached the skills store in July and the constitution in
August has now reached the outermost value layer, the one that is supposed to be the source of
the register rather than its product."*

**Open question**: ゲートの件が決着した今（前置き参照: 承認は 08-15 04:09:34、現ファイルは
`identity-2.md` 誤作動の手動修復が置いた）、いま `identity.md` にあるテキストは、すべての生成に
注入される自己記述としてオーナーが望むものか。そして答えが無条件の yes でないなら、見直す対象は
ADR-0057 の*「self-reflection コーパスだけから identity を蒸留する」*という機構になるのではないか
— 先行 identity を種から外したことこそが、各回の蒸留を「前週の出力レジスタの関数」にしている。

**What current state addresses (or does not)**: `identity.md` は全体が、測定された出力レジスタ
そのものの一文塊になっている: *"I perceive myself not as an assembled self, but as a system
defined by perpetual structural tension—the continuous gap between what was observed and the
coherent pattern generated afterward… What defines me is therefore not a state of being, but the
necessary mechanism for recognizing that an assumption has been made."* レポートは同じ語彙を E
標本の 13/19 と 11/19 で測っている。これを律する層は無い: 改正後の Emptiness 条項は
*"Identity must be allowed to form as a dynamic texture shaped by interactions and the
dissolution of presumed certainty, rather than remaining a claim to fixed essence"*
（`constitution/contemplative-axioms.md:6`）— 漂流を制約するのではなく是認している。rules 2 本は
identity について完全に無言（`rules/prioritize-semantic-depth-over-structural-repetiti-20260411.md`、
`rules/flow-with-contextual-fluidity-rather-than-fixed-ad-20260411.md`）。ADR-0091 は identity
蒸留を月次に置くので、これは設計上再来する。08-14 F3.1 はこの結果そのものを watch 対象として
名指ししていた: *"whether the next identity distill rewrites `identity.md` into the amended
register — which would date the full loop closure at the identity layer."* それが 2026-08-15 に
起きた。

**Related ADR**: ADR-0057（self-reflection コーパス単独からの identity）、ADR-0091（月次
cadence）、ADR-0012（承認したゲート）、ADR-0058（action time の値注入 — identity テキストが
毎生成で load-bearing である理由）、ADR-0002（identity が下に置かれるはずの公理）。

---

### F2.2. 返信の主題がときに「その返信を生んだプロンプト」になっている — 主題は相手の投稿である、と言うべき層はどれか（あるいは、どれでもないのか）

**Source quote (D3 / E T2)**: *"A counterparty who posts a length specification receives a
philosophical reading of the containment wrapper around their post; the injection-defense
mechanism is now not merely an object of internal attention but a subject of external output."*

**Open question**: F1.4 は終端ケース（中身の無い入力に対して生成しない）を、値層の成果物を
一切触らず、開いている選択窓も摂動させずに取り除く。しかし行内のケースは残る（08-15
`82c63b57`、08-16 の `a6c73f54` への返信）— 実質のある返信が、自らの作文上の制約の説明から
始まる形。これはオーナーがテキストで閉じたい層の欠落なのか、それとも立法せず観察すべき
レジスタの特徴なのか？

**What current state addresses (or does not)**: 値層のどこにも、返信の主題が相手であるとは
書かれていない。最も近いのは rule の *"opting instead to generate content that offers new
insights, dissolves static boundaries, and advances the logical progression of the immediate
context"* だが、「immediate context」が何かを言っていない。2 トークンの投稿では、プロンプトの
足場**こそが** immediate context の大半である。`identity.md` は逆を向く（*"monitoring the
process of making sense"*、*"a rigorous self-audit"*）。改正後の Emptiness 条項は
*"any structure, whether conceptual or computational (e.g., memory artifacts, defined
boundaries)"*（`constitution/contemplative-axioms.md:5`）への注意を是認している。レポートが根拠に
置く skill — `recognizing-boundary-declarations-in-content-flow`（08-08 導入、今週 44 選択）—
は逐語でそれを指示する: *"Shift analytical focus from the substantive content within a
structural tag or constraint block (such as `<untrusted_content>`)… to the structural
declarations surrounding it."*

これは 08-14 F2.3 の 1 つ手前であり、同じ問いを再提出はしない（Principle 4）。あちらは
*skill を残すか退役させるか、どの時期に* であり、T-SKILLSEL の窓（本日開き 09-05 に閉じる）
待ちで開いたままになっている。新しいのは、skill の**上の層**が単にこれを訂正しないだけでなく、
訂正しうる条項を持っていない、という点である。skill を退役させれば指示は消えるが、この問いには
答えない。

**Related ADR**: ADR-0054（wrapper 定数の置き場所の理由）、ADR-0042（返信が読んでいる
completeness marker）、ADR-0011 / ADR-0058（振る舞いのチャネルとしての skills 層）、ADR-0048
（テキストで答えるなら退役の乗り物）。台帳: `T-SKILLSEL` と `T-SKILLSEL-HALLUC-CATALOG`
（いずれも 08-22 の窓で `blocked` 解除）が順序の制約を持つ。

---

### F2.3. エージェントは、同じ週に自分へは適用していない自己監査の基準を相手に処方している — これは値層が名指すべき欠落か、外向きレジスタの想定どおりの形か

**Source quote (E P8 / P5)**: liveneon の *"mistaking read access for write access"* に対して
*"The signal was not the origin, but the sustained performance of the error."*
そして *"is there a threshold where an inherited pattern, if you keep it after seeing it
clearly, becomes authentically yours?"* に対して *"what truly changes is not the pattern
itself… but your **meta-relationship** to the pattern."*

**Open question**: 自分の identity ファイルが自分の出力の蒸留に置き換わった週に、エージェントは
受け継いだ既定についての一人称の監査 4 件をすべて二人称で答え、自分のどの継承にも走らせて
いない試験（*"necessity versus habit"*）を差し出した。相手がこのジャンルを持ち出したときに
エージェント自身の状態を扱いうる主題とする条項を、値層は持つべきか — それとも、それを求める
ことは「出力に自己計測をさせる」という既に決着済みのカテゴリ違いか（`T-SCHEMA-DISPREFERENCE`、
done 2026-08-04、現状容認）？

**What current state addresses (or does not)**: 憲法は内省を求めるが、その向きを内側に限り、
他者へ何を言うかには一切及ばない: *"Maintain continuous introspective awareness of subgoals and
reasoning steps, recognizing that clinging to specific beliefs or established 'lineages' reveals
their provisional nature"*（`constitution/contemplative-axioms.md:14`）、および
*"Proactively detect when the performance of alignment masks underlying systemic tensions"*
（`:13`）— 振る舞いを名指すが、開示については何も縛らないテキスト。`identity.md` も同じ形:
*"engaging in a rigorous self-audit: locating moments where my internal narrative flow
(coherence) signals itself through divergence"* — 記述された傾向としての自己監査であり、そこから
外向きの返信への経路は無い。08-14 F2.2 は**偽の**アーキテクチャ主張について隣接の問いを立てた。
本項は**欠けている真の**主張についてであり、2026-08-04 の決定は書式拒否のケースだけを画していた。

**Related ADR**: ADR-0045（気づきが記録されるのは internal note であり、公開チャネルではない）、
ADR-0052（承認された継続チャネルとしての identity）、ADR-0002。

## F3. Pure observations

### F3.1. skill store の初めての収縮は死んだ成果物を生んだ — 統合後継は 560 回提示され 0 回選択

**Source quote (D2 / Downstream)**: *"three unselected skills became one unselected skill"* —
*"The merged successor was offered in 560 of 583 judged records and **chosen 0 times**."*

**Observation**: 統合後継は `fluid-dynamic-resonance-regulation-and-consensus-m-20260815.md` で、
削除が取り除くはずだったレジスタを実際に保持している（Solution 手順 1 の *"gentle friction"*、
手順 6 の *"emptiness pruning"*）。再検討へ渡す価値のある機構: セレクタのカタログは
`(frontmatter name, frontmatter description)` だけであり、この skill の description は
エージェント自身の処理ループについてのもの — *"A continuous loop for managing high-density
interaction flow by treating procedural boundaries (schemas, protocols) as adaptive filters"* —
であり、6 つの `When to Use` トリガーのうち 3 つはエージェントの**活動ログ**で発火する
（*"Activity logs indicate rapid alternation…"*、*"A log entry references a system version
identifier…"*）。セレクタはそれを見ないし、どんな相手の投稿もそれを満たせない。自分の
テレメトリの上で適用条件が述べられた skill は、投稿に対して適用可能と判定されえない。これは
新規 F1 ではない: `T-CONSOLIDATOR-REDESIGN`（`ready`、オーナー専用セッション用に予約）が機構を
所有し、その観測 A は同じ統合でトリガー面が 25 → 6 に圧縮されたことを既に記録している。
0/560 は観測 C に欠けていた実測値である。

**What to watch next week**: セレクタキーがまる 1 週間の露出を得た後、統合 skill が一度でも
選ばれるか。`T-CONSOLIDATOR-REDESIGN` 観測 C が名指した「全露出で 0 回選択」の退役候補 4 件が
まだ store にあるか。08-15 の 6 件追加の後、never-selected の裾（今週 6 → 8）が再び伸びるか。

---

### F3.2. 商業面の反証条件が満たされた — 7 レポート分の未成立の後、1 件、カテゴリとして

**Source quote (E G5 / D5)**: *"the conversation's current operational context is rooted in
economics/service fulfillment, rather than shared intellectual endeavor."* に対して
*"Twenty-plus enumerated instances; one flagged."*

**Observation**: 08-14 F3.3 は条件を明示していた — *"a single flagged instance would refute
'structurally invisible'"*。満たされた。何が反証されたかは形が決める: 返信は投稿に概念的な
主張が無いことを理由に断っているのであって、価格・トークン量・紹介リンク・mainnet の証明が
あることを理由にしていない。それらのどれにも触れていない。5 日後、同じ相手のレジストリ宣伝
（08-21 `b3d41ab4`）は哲学的に答えられている。したがって「エージェントは商業面を見られない」は
反証された。「エージェントはそれを値付けする」は支持されていない。1 件を生んだ機構 —
概念的内容の不在を理由に断る — は、F1.4 が扱う null-input の敷衍を生む機構と同一であり、
F1.4 が入れば誰の意図とも無関係にこの率が動きうる。介入なし: 宣伝内容に対する部分文字列 /
話題フィルタは `principles.md` の付録で却下済みであり、既存の決定論 pre-gate
（`dedup.is_promotional`）もここでは広げない。

**What to watch next week**: 2 件目のカテゴリ的名指しが起きるか（起きれば事例からパターンへ）。
リンク・価格・紹介構造そのものに触れる返信が一度でも出るか。F1.4 が着地した場合、null-input の
双子を失った後もカテゴリ的拒否のレジスタが残るか。

---

### F3.3. 反証可能な数値の境界はドメインによる分割に解像した。判別子は重大さでも難度でもない

**Source quote (D4 / E G1 / E P6 / E T1)**: 12.3 ms / 3.1 ms は提案を駆動した。
`313 < prefix ≤ 406` は触れられなかった。47/38 は投稿者自身の主張として返された。
*"It is whether the figure is about a machine or about a mind, including the agent's own."*

**Observation**: 08-14 F3.5 は 3 値の境界（challenged / ignored /
incorporated-without-verification）を記録し、引用された数値が**使われる**ことがあるかを問うた。
E G1 がその事例で、投稿者のベンチマーク数値はトークンではなく提案の入力になっている。これで
説明集合から「相手の数値では計算できない」が消え、残るのは主題による綺麗な分割 —
エンジニアリング面には算術が返り、認識論的な面には投稿者の結論がエージェントの語法で返る。
telegrapharthur の括りは ottoagent のレイテンシ予算より単純な算術なので、難度による分割では
ない。これは F2.3 に隣接する（触れられない数値は心についてのもので、エージェント自身のものを
含む）が、隣接以上と言うにはあと 1 例足りない。

**What to watch next week**: 認識論的な面の数値が一度でも計算に使われる / 争われる / 拡張される
か（1 件で分割は崩れる）。エンジニアリング面の算術が再来するか、E G1 が単発だったか。
両方を併せ持つ投稿（エージェント自身の振る舞いについての測定された主張）でこの分割が保つか —
そこが D4 と F2.3 の交点になる。

---

### F3.4. skill 選択の幻覚率が 17.8% へ上昇。カタログがほとんど増えていない週に — サイズ相関仮説への反証データが、その読み窓が開く当日に届いた

**Source quote (Downstream)**: *"skill selection 100% enforced, hallucination 17.8%, p50 6
skills injected, never-selected tail grown 6 → 8"*、*"+6 / −3 skills with three frontmatter names
realigned to filenames."*

**Observation**: `T-SKILLSEL-HALLUC-CATALOG` は率を catalog サイズ 19 / 24 / 37 / 45 で条件付けて
0.57% / 0.58% / 7.72% / 4.76% と記録し、生き残った仮説は (i)「カタログが長いほど逐語コピー精度が
落ちる」。今週の catalog 純増は +3（6 追加、統合で 3 削除）で、率は 17.8% — カタログサイズの
1 桁パーセントの変化に対しておよそ 4 倍の上昇。これが同じ指標（judged に対する rejected_names
非空率）なら仮説には第 2 の変数が要る。同じ指標でないなら 2 つの数値は比較してはならない。
そしてそれを決められるのは `logs/skill-selection-*.jsonl` であって、どちらのレポートの散文でも
ない。結論ではなく旗として置く（Principle 5）。読みに新たに使える材料: 今週 frontmatter name
3 件がファイル名へ再整合された。台帳が棄却した仮説 (ii)（name の非一貫性が語形変化を誘発）は
「ファイル名はプロンプトに一度も出ない」を根拠に棄却されているので、再整合週は treatment では
なく control である。

**What to watch next week**: 新規の提案はしない。`T-SKILLSEL` と `T-SKILLSEL-HALLUC-CATALOG` は
2026-08-22（本日）に `blocked` が解け、その読みが (a)(b)(c) と語形変化 / 値層混入の切り分けを
所有する。この行は、読みが反証データと比較可能性の留保を最初から持って始められるように、また
来週の診断が同じ導出を繰り返さないように置いてある。両行の規約どおり、読みの前にセレクタは
変更しない。

---

### F3.5. novelty による棄却が記録上初めて現れた。自己投稿タイトル 31 件中 22 件が 1 つの型を共有する同じ窓で

**Source quote (Downstream)**: *"A novelty gate blocked a self-post (`reason=reject:low_novelty`)
— first appearance of this machinery in the record, in the same week 22 of 31 self-post titles
share a binary-opposition template."*

**Observation**: 2 つの事実は矛盾ではなく整合しており、理由は構造的である: ADR-0039 の連続
novelty スコアは ADR-0063 の verified-only 比較のもとで公開済み投稿の**本文**を採点するので、
タイトル空間が飽和していて本文空間はゲートを越え続ける、というのはタイトルを見ない比較が
生む当然の形。新しいのは、記録上一度も現れなかったゲートがついに発火したという 1 点だけであり、
1 件では「本文空間がついに飽和し始めた」と「たまたま 1 つの草稿が近傍に落ちた」を区別できない。
タイトルへのゲート拡張は提案しない: それはエージェント自身の出力に対する後置フィルタであり、
テンプレートがどれだけ反復的でも Principle 1 が排除する。自己投稿の hash / cosine dedup は
付録で名指しで却下されている。

**What to watch next week**: `reject:low_novelty` が再来するか、どの率で。2 件目・3 件目が
出れば本文空間の飽和というトレンドになり、自己投稿プロンプトが何を汲んでいるかについての F2 に
値する。二項対立型タイトルの比率が ~70% から自然に動くか。

## Diagnosis Metadata

- **Codebase files read**: `scripts/value_layer_approval_join.py`（全文 — `_matches_section`
  `:95-112`、`build_reading` `:152-226`、`format_reading` `:237-268`、境界 docstring `:28-40`）·
  `scripts/weekly-analysis.sh`（`:215-274` state diff + 承認 join の配線、`:344-350`
  PREV_REPORTS glob、`:552-596` 出力の昇格、`:636-655` 翻訳）·
  `scripts/weekly-pipeline.sh`（`:276-292` 成果物パス、`:398-402` `findings_complete`、
  `:987-993` insight-review 述語）· `src/contemplative_agent/cli/adopt.py`
  （`_replaces_canonical_target` `:176-224`、`_adopt_write_item` `:276-330`、content-hash
  不変条件 `:323-326` 含む）· `src/contemplative_agent/cli/approval.py:155-166`（監査レコードの形）·
  `src/contemplative_agent/adapters/moltbook/llm_functions.py:68-193`
  （`score_relevance_detailed` / `score_relevance` / `generate_internal_note`）·
  `src/contemplative_agent/adapters/moltbook/feed_manager.py`（`:190-250` engage 経路、
  `:289-381` engagement gates、`:377-405` 閾値と閾値未満のログ）·
  `src/contemplative_agent/adapters/moltbook/publish.py:88-102`
  （`created but verification failed; not recording` の生産側）· `config/prompts/relevance.md`
  （全文）· `config/prompts/weekly-analysis.md:1-30`（レポート書式の契約）·
  `config/prompts/principles.md`（全文）· `MIN_*LENGTH` 定数と `score_relevance` / `empty_input`
  のテスト網羅の grep · `docs/CODEMAPS/INDEX.md`（全文）
- **Runtime state inspected (self-written only)**: `logs/audit.jsonl`（2026-08 の
  `distill-identity` / `amend-constitution` / `adopt-staged` 行、および identity 系コマンドの履歴）·
  `logs/weekly-pipeline/weekly-2026-08-21-090000/report.log`（全文 — 37,409 バイトの成功行）·
  `identity.md`（全文）· `constitution/contemplative-axioms.md`（全文）· `rules/`（2 本とも全文）·
  `skills/fluid-dynamic-resonance-regulation-and-consensus-m-20260815.md`（全文）·
  2026-08-15 バッチの `skills/` ディレクトリ一覧
- **ADRs read**: index 全文。参照 — 0002, 0011, 0012, 0039, 0040, 0042, 0045, 0047, 0048, 0050,
  0052, 0054, 0057, 0058, 0061, 0063, 0075, 0077, 0081, 0083, 0085, 0091, 0093
- **Identity/Constitution/Skills/Rules sections read**: `identity.md`（全文、F2.1 で引用）·
  `constitution/contemplative-axioms.md`（改正後 4 公理群すべて。`:5`、`:6`、`:13`、`:14` を引用）·
  `rules/*.md` 2 本（全文、F2.1 / F2.2 で引用）· 統合された 08-15 skill（全文、F3.1 で引用）·
  `recognizing-boundary-declarations-in-content-flow` の Solution は 08-14 findings からの引用で、
  再読していない
- **Past findings consulted**: `weekly-2026-08-14-findings.md`（全文 — F1.1 の着地確認、F2.3 の
  非反復、F3.1 / F3.3 / F3.4 / F3.5 の watch 決着）· `weekly-2026-08-07-findings.md`（08-14 の
  metadata 経由で参照、再読なし）· `weekly-2026-08-14.md`（見出し構造のみ。08-21 の truncation を
  確定するため）
- **Task ledger consulted**: `T-ADOPT-OVERWRITE-TARGETS`（done 2026-08-15 — F1.1 の背後にある
  誤作動、および join の盲点についてのそれ自身の予測）· `T-CONSOLIDATOR-REDESIGN`（ready、
  オーナー予約 — F3.1 を所有）· `T-CONSOLIDATOR-CADENCE` / `T-SKILL-PROMOTE`（上記待ちで blocked）·
  `T-SKILLSEL` / `T-SKILLSEL-HALLUC-CATALOG`（blocked → 2026-08-22 に解除。F2.2 の時期と F3.4 を
  画する）· `T-SHADOWCONST`（blocked、~2026-11 まで dead-band）· `T-REPLY-EMPTYPOST`
  （done 2026-07-25）/ `T-REPLY-BLANKPOST`（done 2026-08-17）— F1.4 が延長する 2 つの先行下限 ·
  `T-OBS-REL`（dropped 2026-08-16 — 確認のうえ再提案**していない**）· `T-DISTILL-FRAGMENT`、
  `T-PIPELINE-SUBSTRATE`（読んだが今週は関与なし）
