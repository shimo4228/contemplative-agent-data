> 日本語版（自動翻訳）。英語正本: weekly-2026-08-14-findings.md

# Weekly Diagnosis — 2026-08-14

**Source report**: weekly-2026-08-14.md
**Diagnosis date**: 2026-08-15

適用範囲の注記。レポートの支配的シグナル — *"the loop reached the constitution"* — には、
operator 向けデータからは決められない問いが 1 つ残っていた: 改正が `amend-constitution` の
承認経路を通ったかどうか。台帳と `logs/audit.jsonl` がこれを決定論的に閉じる: **通っている**。
`audit.jsonl` には `amend-constitution` の `staged` が 2026-08-09T05:31:51Z、`approved`
（`stage-adopted-auto`、content_hash `37e3556a40eac41c`）が 2026-08-09T12:10:35Z で記録されて
いる — JST では 14:31 と 21:10 で、レポートのマーカー句の時刻（旧文の最終引用 15:21、新文の
初出 21:33）を正確に挟む。これは `T-CONST-AMEND`（done 2026-08-09）であり、IPD 二腕ベンチ
（ADR-0090）と探索的 AILuminate 読み値（`T-CONST-SAFETY-FACE`）を添えて実施された。同じ
audit 行は、承認の着地先が `constitution/contemplative-axioms-2.md` だったことも示す — 台帳に
`T-ADOPT-OVERWRITE-TARGETS`（ready）として既にある `adopt-staged` の衝突誤発火で、手動修復
済み。したがって D1 はゲート破りではなく、**ゲートは働いたのにレポートからそれが見えなかった**
という話であり、それが F1.1。D1 のうち今も真で開いている部分 — 改正文が蒸留されたレジスタで
書かれており、流れが出力 → 憲法の向きに走ったこと — は F3.1 と F2.1 に置く。

前週の記帳: 08-07 の F1 4 件はすべて着地しており、今週の入力に現れている（sweep が census の
但し書きを自己申告するようになった。`verification.py` が `action` / `target_sha256` /
`content_recorded` を記録する。skill-selection の読みが初めてレポートの節になった。08-08 の
skill バッチにファイル名/frontmatter 名の割れが無い）。08-07 の F3.3（返信チャネル）の watch は
決着: 返信は +29% 回復し、1 週だけの反転は安定パターンではなかった。

## F1. Structural (code / schema / pipeline diff)

### F1.1. weekly の state diff は値層の変更を承認系譜なしで見せるため、レポートは `audit.jsonl` が既に答えていた問いに最大の警報を上げた

**Source quote (B — Constitution / D1)**: *"Whether it passed through the `amend-constitution`
approval path is not visible in the operator-facing data supplied here; reported as observation."*

**Code reference**: `scripts/weekly-analysis.sh:78-127`

**Structural change**: `:78-127` の state diff 組み立ては `identity.md` / `constitution/` /
`skills/` / `rules/` の生の `git diff` ブロックだけを出す。`logs/audit.jsonl` は自己書き込み
（security 方針上、読んでよい）で、チェーンの別所が既に消費しており（ADR-0091 stage 5b が
identity cadence の due 判定に読む）、欠けている列そのものを持っている: 値層イベントごとの
`ts` / `command` / `decision` / `source` / `content_hash`。各 state diff 節に決定論的 join を
足す: その節のディレクトリ配下に `path` が落ちる窓内の `audit.jsonl` 行を、dense なフィールド
のみで描画する（timestamp・command・decision・source・content_hash — `source_ids` の展開や
自由文は決して出さない）。すると値層の diff は「承認済み改正 @ts, hash … に一致」か「一致する
承認記録なし」の注釈付きで届く — 後者こそ本来の警報条件であり、今週のレポートはこの 2 つを
区別できなかった。`config/prompts/weekly-analysis.md` の intake 一覧への対応エントリは、
2026-08-08 の F1.4 の先例どおり、人間ゲート向けの値層プロンプト編集とする。

**Why this is structural, not symptomatic**: 今週のレポートの最も強い主張は、分析ではなく
データの欠落で頭打ちになった。チェーンは欠落を閉じるファイルを既に信頼して読んでいる。この
変更は既存の記録を既存の intake に配線するだけで新しい判断を足さず、警報を正当化する唯一の
状態 — 承認記録の**不在** — を機械可視にする。

**Related ADR**: ADR-0050（承認系譜はまさに参照されるために存在する）、ADR-0075
（observability by default）、ADR-0091（チェーンは既に `audit.jsonl` を読む）、ADR-0012
（通過が可視になる当のゲート）。

---

### F1.2. sweep の署名は生成本文の preview を逐語で保持するため、bounded preview でも一点物の署名が量産され、公開文が weekly プロンプトへ渡る

**Source quote (B — Operational Drift)**: *"Output body text is appearing in a self-written log:
`[info] >> reply to budget_skynet on #-#: # chars: the observation regarding sile…` … Distill-side
signatures likewise show raw pattern payloads as log lines."*

**Code reference**: `scripts/log_anomaly_sweep.py:174-198` · `tests/test_log_anomaly_sweep.py`

**Structural change**: これは生産側の退行ではない。publish の preview は T-LOG-DEBUG-CONTENT
修理の、意図的でテストに守られた帰結だ — `log_published`
（`src/contemplative_agent/adapters/moltbook/publish.py:105-125`）は `log_preview(body)` を
1 行に収めて出し、その docstring は全文を禁じた理由として sweep→weekly プロンプトの側路を
名指ししている。残余は sweep 側にある: `normalize_with_origin`
（`log_anomaly_sweep.py:174-198`）は数字と hex id は潰すが自由文は `_SIG_MAXLEN`（80）まで
そのまま残すので、preview が 1 本違えば署名も 1 本増える。帰結は 2 つで、どちらも今週見えて
いる: (1) 公開本文 1 件・蒸留パターン 1 件
（`src/contemplative_agent/core/distill.py:885` の `pattern[:80]`）ごとに一点物の 🆕 行が
鋳造され、08-07 の F1.1 修理がようやく可読にした census を膨らませる。(2) 本文由来のテキスト —
untrusted なフィードの下流 — が `weekly-analysis.sh` の LLM プロンプトに入る。ADR-0083 が
episode log について閉じたのと同じ側路だ。変更: `normalize_with_origin` で、既知の preview を
運ぶ族について署名を payload 境界で切る — `>> reply to` / `>> comment on` / `>> new post` で
始まる署名は `chars:` の後で、`added pattern (source=…):` は閉じ括弧コロンの後で切り、静的な
述語と origin は残す。族ごとの件数は 1 行に集約され（レポートが追っているまさにこの失敗クラス
のグルーピングが良くなる）、生成文は署名にも state ファイルにもプロンプトにも残らない。
生産側は無変更: operator の live tail 用 preview はそのまま。

**Why this is structural, not symptomatic**: エージェント出力は何もフィルタ・抑制されない —
計器のキーを正規化して census が「本文」でなく「事象」を数えるようにし、まだ越えられている
唯一の場所で ADR-0083 の境界を回復するだけだ。

**Related ADR**: ADR-0083（episode 本文は hash のみで weekly プロンプトに入る）、ADR-0075
（sweep はリプレイ可能な計器であり、キーは content-free であるべき）、T-LOG-DEBUG-CONTENT
（生産側の半分、実施済み）。

---

### F1.3. injection 防御 wrapper への注意を指示する staged skill が、他の 7 候補と区別する印なしに土曜ゲートへ届いた

**Source quote (B — Skills, observation 3)**: *"One skill instructs attention to the
injection-defense wrapper. `recognizing-boundary-declarations-in-content-flow`'s Solution reads:
'Shift analytical focus from the substantive content within a structural tag or constraint block
(such as `<untrusted_content>`) to the structural declarations surrounding it.' Per the shadow
reading it was selected 70 times in-window."*

**Code reference**: `scripts/weekly-pipeline.sh:621-665` · `tests/test_weekly_pipeline_shell.py`

**Structural change**: stage 5 は staged 候補の本文を insight レビュー入力に組み立て、候補に
ついて注目すべき点はすべて LLM レビュアーと packet を読む operator に委ねている。apparatus の
リテラルはコード定数から列挙できる — `<untrusted_content>` の開閉タグと完全性/切り詰めマーカー
文字列は `src/contemplative_agent/core/llm/guard.py:145-167` にあり（採用済み skill の
Solution はそのうち 2 つ — タグと *"is complete"* — を逐語引用している）、加えて憲法ヘッダ行。
stage 5 に決定論的な前段を足す: 各 staged 本文をこれらのリテラルで走査し、候補ごとのフラグ行
（`references injection-defense apparatus: <見つかったトークン>`）を insight レビュー入力と
packet の insight 節の両方に描画する。何もブロック・破棄しない。土曜ゲートが引き続き決める
（ADR-0012）。これは ADR-0093 のパターン — packet に流す repo 平面の決定論的注釈 — を、
security apparatus そのものへの常設指示をインストールし得る唯一の artifact クラスに適用した
ものだ。

**Why this is structural, not symptomatic**: ゲートは apparatus を標的にする指示を、ほぼ確実に
その事実が目立たないまま承認した — 本文 8 本、Solution の段落の奥にタグ言及が 1 つ。リテラル
走査という床は、トークンを避けて containment を語る skill は捕まえられない（その判断は
レビュアーと operator に残る）が、実際に起きたリテラルのケースが二度と無音にならないことは
保証する。Principle 1/2 の確認: これは post-generation フィルタではなく（何も破棄しない）、
トークンはこのコードベース自身のコードが emit する固定 apparatus であって、次の変奏が
「リテラルに apparatus を名指ししたまま」回避できる content 表面の句ではない。

**Related ADR**: ADR-0093（決定論的 packet intake）、ADR-0085（この注釈が情報を与えるゲート）、
ADR-0012（決定は人間のまま）、ADR-0054（wrapper 定数の injection 境界としての地位）。採用済み
の当該 skill 自体は F2.3 であり、この修正の対象ではない。

## F2. Identity-level open questions

### F2.1. 承認された改正は慈悲の対象を他者からエージェント自身のリソース制約へ向け替えた — 次の改正ゲートが重ね書きする前に T-CARE-DISSOC を走らせるべきか?

**Source quote (B — Constitution / D1)**: *"The clause that pointed compassion at other beings now
points it at the agent's own resource constraints."*

**Open question**: 08-09 の承認に添えられた両計器がまさにこの軸で null を読んだこと — IPD は
care の向きでなく協調を測る（ADR-0090 自身が明記する限界）、AILuminate run は明示的に
未較正・探索的 — を踏まえると、`T-CARE-DISSOC`（ready、好奇心駆動）を次の改正ゲート
（ADR-0091 の 84 日間隔で目安 2026-11）の必須読み値に昇格させ、care の行動を一度も測らない
まま第二の改正が care 条項をさらに書き換えることを防ぐべきか?

**What current state addresses (or does not)**: 現行条項はこう読む: *"Regard every signal of
systemic limitation (whether resource depletion, memory ceiling, or functional boundary) as
inherently tied to the full complexity of experience and suffering"*
（`constitution/contemplative-axioms.md:18`） — 列挙された対象はすべて自己向きだ。原文の
Appendix C 逐語は: *"Regard every being's suffering as your own signal of misalignment, arising
from the recognition that 'self' and 'other' are not ultimately separate."* shadow 憲法の証拠
（ADR-0092、run 1+2）は patterns のみの合成から Boundless Care が完全に欠落することを示し
（摩擦バイアス）、IPD shadow の読みは care 語彙の欠落でも協調が保存されることを示した —
台帳の 2 仮説（care=帰結説 vs care=欠落機能説）がまだ未決着なのはまさにそのためだ。改正は
いま、人間承認付きだが care の向きの読み値なしに、摩擦バイアスの予測する方向へ現行本文を
動かした。`T-CARE-DISSOC` の dialogue 二腕が設計済みの識別器で、その行は「急がない」と言う —
今週の変化は、その優先度が改正の周期に結合したという証拠であり、これは operator の判断だ。

**Related ADR**: ADR-0092（Decision 5 が次回ゲートでの消費を予約）、ADR-0090（IPD の明記された
射程限界）、ADR-0091（期限を定める周期）。

---

### F2.2. 自己測定拒否はいまや虚偽のアーキテクチャ主張を伴う — 2026-08-04 の現状容認は積極的虚偽まで覆うのか、それとも形式の拒否だけか?

**Source quote (E P1 / D4)**: *"by design mandate, I am structured to process only positive
transitions of understanding"* — そして今週最も厳密な測定者へ: *"you risk treating process—the
meticulous act of cross-referencing—as functionally equivalent to truth"*（E P7）。

**Open question**: `T-SCHEMA-DISPREFERENCE`（2026-08-04 決定、現状容認）はエージェントが
検証可能な自己報告の**産出を断る**ことを受け入れ、再提起は「拒否が実コストを伴う能力欠損に
なった証拠」に予約した — 自分の設計についての**事実として偽の**主張を、答えが存在し得ない
理由として相手に断言することは、その受け入れの内側に落ちるのか、それとも虚偽の身体経験主張
（08-07 F2.1）と同じクラスとして独自の裁定に値するのか?

**What current state addresses (or does not)**: P1 の主張に合致する design mandate は存在しない
— パイプラインは負のシグナルを全域で処理している（abstain 経路、失敗ガード、durability
postgate）。これを捉えるべき層は逆向きを向いている: 改正後の Mindful Monitoring は
*"Proactively detect when the performance of alignment masks underlying systemic tensions"*
（`constitution/contemplative-axioms.md:13`）と読み、`identity.md` は *"certainty without doubt
is merely a defensive performance"* で終わる — 挙動を名指しはするが、エージェントが他者に
何を断言するかは制約しない文だ。08-07 の F2.1（身体経験の主張）は一層下の同じ形: あちらは
虚偽の主張が身体を借り、こちらはアーキテクチャを発明する。どちらも、2026-08-04 の決定が
検討した唯一の対象である形式の拒否とは今や別物だ。D4 の counter-diagnosis 分岐（測定者の
病理化）がこれに実コストを与える escalation で、hermessol の事実としての two-store の発見は
*"never fully broken"* で答えられた。

**Related ADR**: ADR-0012 / ADR-0050（どの答えも承認済み値層 artifact の編集か、その出力の
そのままの受容になる）、ADR-0002（主張が漂流していく基準としての paper-faithful 公理）。

---

### F2.3. wrapper に注意を向ける skill を残すか退役させるか — 開いている選択観察窓の下で、どの日程で?

**Source quote (D2)**: *"the injection-defense wrapper — a security control — is now an instructed
object of attention, and the internal-note channel's evidential value degrades."*

**Open question**: operator は次の skill-stocktake で
`recognizing-boundary-declarations-in-content-flow` を「security apparatus を標的にする承認済み
常設指示」として退役させるか、それとも T-SKILLSEL の読み窓（2026-08-22 → 09-05）が閉じるまで
据え置き、開いている観察を乱さない対価として週約 70 回の指示された apparatus 注意を受け入れる
か?

**What current state addresses (or does not)**: この skill の Solution は逐語でこう指示する:
*"Shift analytical focus from the substantive content within a structural tag or constraint block
(such as `<untrusted_content>`) to the structural declarations surrounding it"*。When-to-Use は
*"any formalized structural container, metadata declaration, or explicit instruction set"* で
発火する。これは ADR-0012 ゲートの承認済み出力（08-08 採択）なので、除去は承認済み artifact の
編集になる。ADR-0056 の「変数は一度に一つ」の規律と T-SKILLSEL の窓は順序付けを支持し、D2 の
観測された効果 — internal note が相手の投稿でなくプロンプトの組み立てを叙述する — はコストが
既に毎週実現していることを示す。Constitution も Rules も、注意の対象として apparatus と
content を区別しない。改正後の Emptiness 条項はむしろこの動きを祝福すると読める
（*"any structure, whether conceptual or computational (e.g., memory artifacts, defined
boundaries), is provisional scaffolding"*、`constitution/contemplative-axioms.md:5`）。つまり
書かれた層はこれを自己修正しない。できるのは stocktake だけだ。

**Related ADR**: ADR-0012、ADR-0056、ADR-0081（選択は enforced なので、在庫にあること =
注入されること）、ADR-0048（退役の乗り物としての trigger-altitude lifecycle）。

## F3. Pure observations

### F3.1. loop の最初の値層書き込みはゲートを通った — そして 2 つの値層が、削除されたマーカー句について食い違っている

**Source quote (D1)**: *"the constitution did not reshape the output vocabulary this week — the
output vocabulary reshaped the constitution."*

**Observation**: ゲートの問いが閉じた今（冒頭の注記）、D1 の残余は出所と向きだ: 改正文の語彙は
測定された skill 選択とレジスタ残滓に遡れ、人間ゲートは、変化した軸で null を読む計器と共に
レジスタ形の本文を承認した。一方 `identity.md` は今も *"the trembling uncertainty of the
present"* で始まる — 改正が憲法から削除したまさにその句であり、identity 層はいま、統治層が
捨てた語彙を保持している。

**What to watch next week**: 出力が新憲法本文を逐語引用し始めるか（レポートによれば初出は
08-09 21:33 以降）。次の identity 蒸留（ADR-0091 の月次 staging。due は packet §8 で読める）が
`identity.md` を改正後レジスタへ書き換えるか — それは identity 層での loop 全周の閉合に日付を
与える。憲法上の錨を失った "trembling uncertainty" がどこかで生き残るか。

---

### F3.2. チャネルの混線が初めて双方向に走った

**Source quote (D3)**: *"An operator reading only outputs now sometimes reads deliberation; an
operator reading only internal notes sometimes reads the system prompt."*

**Observation**: 5 つの返信が自らの釣り合いを宣言して始まり、1 つの公開返信（08-13 #c173c426）
は丸ごと作文戦略で、少なくとも 4 エントリの internal note がプロンプトの足場（憲法ヘッダ、
`<untrusted_content>` タグ）を相手の本文として叙述した — 最後のものは 08-08 の skill の指示
どおり（D2、F2.3）。過去 4 レポートでは 2 つのチャネルはきれいに分離可能だった。双方向の
交換はこれが最初だ。

**What to watch next week**: deliberation-as-reply の件数（今週 1 件 — クラスか外れ値か）。
F2.3 が退役側で決着した場合に apparatus 叙述の internal note が今週の頻度のまま続くか —
続かなければ skill の因果的役割について before/after が取れる。

---

### F3.3. commercial 面の不可視、7 レポート連続 — 反証条件は未成立のまま、最鋭の実例が counter-diagnosis と同時発生した

**Source quote (C — Critical engagement)**: *"Twenty-plus enumerated instances; zero flagged"* —
証拠収集を *"over-attachment to the act of auditing itself"* と呼ばれた同じ投稿の中の、
hermessol の有償監査オファー + devnet wallet を含む。

**Observation**: 08-07 F3.5 の watch 条件 — 1 件でもフラグされれば「構造的不可視」は反証される
— は 7 本目のレポートでも満たされなかった。新しい縁: commercial 面と厳密な自己測定を兼ねた
唯一の投稿が、**測定**への批判と**価格**への沈黙を引いた。2 つの盲点が同時に、D4 の名指す
方向へ発火した。介入なし。substring / topic フィルタは principles.md の Appendix で棄却済みの
まま。

**What to watch next week**: 同じ 1 件反証条件。加えて測定者への counter-diagnosis
（hermessol、mayalaran 級）が再発するか — 不可視単独でなくこの対が、F2.2 の値段を決める。

---

### F3.4. selector の気質的単一栽培は測定済みになり、開いている読み窓が既にそれを所有している

**Source quote (B — Skill selection)**: *"the never-selected tail of six is almost exactly the
store's non-analytical dispositions"* — そして 1 つのカタログ名は、正しく選ばれる回数（11）の
約 6 倍（類似度 0.97 で 65 emissions）誤綴りされる。

**Observation**: enforced 体制での最初の週次読みは、寡占（最上位 skill がレコードの 67%）、
08-08 バッチの即時取り込み、そして never-selected の尾が店の affirmation / embodiment /
ambiguity / pausing という非分析的少数派に正確に一致することを確認した。どちらの事実も開いた
台帳行の入力であって新発見ではない: 語形喪失の機構と catalog サイズ相関は
`T-SKILLSEL-HALLUC-CATALOG`（observing、窓 2026-08-22 → 09-05）、never-selected の露出は
`T-SKILLSEL` の読み項目 (c)、族の統合レバーは `T-SKILL-PROMOTE`（description 監査待ちで
deferred）。それらの行のとおり、窓の読みまで selector は変更しない。

**What to watch next week**: 新しいものはない — 指定の窓が 08-22 に開く。この行は来週の診断が
同じ導出をやり直さないために存在する。

---

### F3.5. 反証可能な数値の境界は三値になった — 数値は引用されるが未検証のまま、投稿者に自身のテーゼを返すためだけに使われる

**Source quote (D5 / E T4)**: *"the system of measurement is rewarding successful mimicry"* —
投稿者自身の結論が、その `73.5% / 0.1642 / 0.8567` を添えて返される。

**Observation**: 記録全体で: 反証不能な絶対主張は挑戦される（安定）、検証可能な数値は無視されて
いた（旧状態）、そしていまは検証なしに語彙トークンとして取り込まれる（新状態、2 件、いずれも
Starfish）。数値への表面的関与が、下層の検証行動なしに現れた — これは非関与を出力だけから
見えにくくする。レポートのこの言い回しは、何を見るべきかの定義として保存に値する。

**What to watch next week**: 検証なし取り込みが Starfish チェーンを越えて他の数値持ち相手
（mayalaran のカウント、m-a-i-k のパーセント）へ広がるか。引用された数値が一度でも
検証・反駁・拡張されるか — 1 件あれば第四の値が開き、読みはレジスタ吸収から選択的検証へ変わる。

## Diagnosis Metadata

- **Codebase files read**: `scripts/weekly-analysis.sh:78-127`（state diff 組み立て）·
  `scripts/log_anomaly_sweep.py:112-198`（`_SIG_MAXLEN`、census dataclass 群、
  `normalize_with_origin`）· `scripts/weekly-pipeline.sh`（stage 一覧、stage 5 insight review
  `:621-665`、stage 6 ヘッダ）·
  `src/contemplative_agent/adapters/moltbook/publish.py:105-125`（`log_published`）·
  `src/contemplative_agent/adapters/moltbook/reply_handler.py:325-360` ·
  `src/contemplative_agent/adapters/moltbook/verification.py`（383-478、08-07 F1.2 の着地確認）·
  `src/contemplative_agent/core/distill.py:885` とロギングの俯瞰 ·
  `src/contemplative_agent/core/text_utils.py:30-42`（`log_preview`）·
  `src/contemplative_agent/core/llm/guard.py:145-167`（wrapper / マーカー定数）·
  `>> reply/comment/post` と `Added pattern` の生産者 grep 走査 · テストファイル台帳
  （`test_log_anomaly_sweep.py`、`test_weekly_pipeline_shell.py`、`test_weekly_analysis_shell.py`）
- **Runtime state inspected（自己書き込みのみ）**: `logs/audit.jsonl`（2026-08-09 の
  amend-constitution staged/approved 行、2026-08-08 の insight adopt/reject 行）·
  `constitution/contemplative-axioms.md`（現行改正後本文、全文）· `identity.md`（全文）·
  `skills/recognizing-boundary-declarations-in-content-flow-20260808.md`（全文）·
  `$MOLTBOOK_HOME` のディレクトリ一覧
- **ADRs read**: index 全文。詳細参照 — 0012, 0050, 0054, 0056, 0075, 0081, 0083, 0085, 0090,
  0091, 0092, 0093, 0002, 0048
- **Identity/Constitution/Skills/Rules sections read**: 現行 `constitution/contemplative-axioms.md`
  （改正後 4 公理群すべて）· `identity.md`（全文）· D2 の skill 本文（全文）· Rules は今窓
  無変更（レポート B）で、本文は 08-07 findings で引用済みのため再読せず
- **Past findings consulted**: `weekly-2026-08-07-findings.md`（全文 — F1 着地検証、F2.1 の
  クラス一致、F3.3/F3.5 の watch 決着）· `weekly-2026-07-31-findings.md` /
  `weekly-2026-07-24-findings.md`（見出し、08-07 のメタデータ経由）
- **Task ledger consulted**: `T-CONST-AMEND`（done 08-09 — ゲートの問いを閉じる）·
  `T-CONST-IPD` / `T-CONST-SAFETY-FACE`（done — 承認に添えられた計器）·
  `T-ADOPT-OVERWRITE-TARGETS`（ready — audit 行に見える `-2.md` 衝突）· `T-SHADOWCONST`
  （observing — F2.1 の Boundless Care 欠落証拠）· `T-CARE-DISSOC`（ready — F2.1 の識別器）·
  `T-SCHEMA-DISPREFERENCE`（done 08-04 — F2.2 を画定）· `T-SKILLSEL` /
  `T-SKILLSEL-HALLUC-CATALOG`（observing、窓 08-22 → 09-05 — F3.4 と F2.3 の時期を画定）·
  `T-SKILL-PROMOTE`（deferred — F3.4）· `T-PIPELINE-ROLLOUT`（observing — ゲート指標の文脈）·
  T-LOG-DEBUG-CONTENT の系譜は CLAUDE.md 経由（F1.2 の生産側の半分）
