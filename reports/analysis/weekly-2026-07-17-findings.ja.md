# Weekly Diagnosis — 2026-07-17

> 日本語版（自動翻訳）。英語正本: weekly-2026-07-17-findings.md

**Source report**: weekly-2026-07-17.md
**Diagnosis date**: 2026-07-18

> **Scope note.** 上流レポートには見えなかった 4 点が、今週のシグナルの読みを変える:
>
> 1. **レポート自体が今日、劣化状態で生成された。** 09:00 の launchd 実行は Claude の
>    セッション上限に当たりエラー 1 行を書いて終了。10:20 の再生成は、失敗ランが既に
>    消費した state に対して anomaly sweep を再実行した。**今週の sweep の Δ / 🆕 列は
>    週次ではなく約 80 分の窓（09:00 → 10:21）を測っている** — しかもその窓は 08:00 に
>    走り出した insight run の実行中活動に支配されていた（→ F3.1）。
> 2. **「No previous report provided」はファイル不在ではなくスクリプトのバグ。**
>    過去 3 週分のレポート（07-11, 07-05, 06-28）は存在するが、過去レポート探索は
>    ちょうど −7/−14/−21 日のファイル名（07-10, 07-03, 06-26）を計算して探すため
>    1 件も見つからない。Principle 4 のガード基盤が上流プロンプトから静かに欠落した
>    （→ F1.2）。
> 3. **自己スレッド形成（D3）には決定論的な原因がある。** 自己識別ゲートの全箇所 —
>    own-post skip、2 経路の自己コメント skip、自己投稿シードの除外 — が、live
>    platform が供給しない author id でキーされている。ゲートは設計上存在するが
>    一度も発火したことがない（→ F1.1）。
> 4. **insight パイプラインのシグネチャ（+5 title-drop / +5 extraction-failed）は
>    2026-07-18 の novelty gate 容量失敗**であり、既に診断・台帳化済み
>    （T-INSIGHT-NOVELTY / T-INSIGHT-C12 / T-INSIGHT-OBS）。ここでは何も追加提案しない。
>
> 先週の F1.1（ログチャネル分離）は窓内に `a56b167` として出荷済み — 運用ログには
> もう生成本文の全文は流れない。

---

## F1. Structural (code / schema / pipeline diff)

### F1.1. 自己識別ゲートはすべて「live feed が供給しない author id」と比較している — own-post skip と自己コメント skip は構造的に死んでいる。author name でキーし直す

**Source quote (D3; C — Self-reference; B)**: *"07-12 #3e670ab6,
07-13 #ae54f138, 07-16 #9e377077 — contemplative-agent commenting on / replying
to contemplative-agent, with the later reply 'This resonates sharply with a
concern raised in the third piece.'"*; sweep 常在シグナル
`self-reply protection degraded` ×69。

**Code reference**:
- `src/contemplative_agent/adapters/moltbook/feed_manager.py:157-160` — `author_id = author.get("id", "")` とコード内コメント *"Live feed posts carry author.name but typically not author.id"*
- `src/contemplative_agent/adapters/moltbook/feed_manager.py:266-268` — own-post ゲート: `if ctx.own_agent_id and author_id == ctx.own_agent_id`
- `src/contemplative_agent/adapters/moltbook/reply_handler.py:217-220` — 通知経路の自己コメント skip: `replier_id == self._ctx.own_agent_id`
- `src/contemplative_agent/adapters/moltbook/reply_handler.py:399-401` — コメント走査経路の自己コメント skip: `fields["agent_id"] == self._ctx.own_agent_id`
- `src/contemplative_agent/adapters/moltbook/post_pipeline.py:194-206` — 自己投稿シードの除外、同じ id キー
- `src/contemplative_agent/adapters/moltbook/agent.py:148-152` — `/home your_account.name` はローカル変数に取り出してログに書くだけで**捨てている**。`session_context.py:33-52` に own name のスロットは無い
- `src/contemplative_agent/adapters/moltbook/agent.py:177` — `"Self-reply protection DEGRADED: own agent ID unknown"`（sweep の ×69 の出所）

**Structural change**: 自分の *name* を保持し、4 箇所のゲートで name も比較する。
具体的には `SessionContext` に `own_agent_name` を追加し、`_fetch_home_data` で
`your_account.name`（既に手元にある値）から設定する。各ゲートは
`author_id == own_agent_id` **または** `author_name == own_agent_name` で skip
（id 比較は将来 API が id を返すようになった場合の保険として残す）。相手側の
name は全経路で既に抽出済み（`feed_manager.py:160`、`extract_agent_fields` /
`extract_notification_fields`）。

**Why this is structural, not symptomatic**: 新しいフィルタを足すのではなく、
設計上存在するゲートを修理する。生成された出力は一切検査しない（Principle 1 に
非抵触）し、トークンも指名しない（Principle 2 — 自分のアカウント名は自己識別で
あってコンテンツ標的ではない）。これは ADR-0055 自身の re-key の完遂にあたる:
ADR-0055 は 271/271 レコードで `agent_id="unknown"` を確認して author-history
ゲート群を `agent_name` キーへ移したが、自己識別ゲートは移し残された（ADR-0059
が `get_history_with` について記録したのと同じ「not carried along」パターン）。
D3 の閉ループ — 自分の過去の reframe を「外部のもののように」引用する — は
この下流にある: ゲートが死んでいるため自分の投稿が engagement 候補として、
自分のコメントが返信対象として再流入し、framework の出力が構造的に framework の
入力になる。受容するリスクは ADR-0055 と同一（表示名衝突、bug-audit 2026-07-06
M6）: 「contemplative-agent」を名乗る別 agent を skip してしまう — author-history
ゲートで既に受け入れたトレードと同じ。

**Validity self-check**: 未実装（4 箇所とも 2026-07-18 に実読。name キーの自己
チェックは存在しない — `own_agent_id` の grep は id 比較のみ、own name は保存
されていない）; パラメータが既に有効ということもない（`feed_manager.py:292-296`
のコード内 docstring が id キー版ゲートは「never fired」と明言）; 棄却 ADR なし —
ADR-0055 はむしろ先例であり、ADR-0059 の削除対象は会話的 reply *history*
（ADR-0052 の continuity channel 抵触）であって、cross-session コンテンツを
運ばない自己識別は対象外; principles.md の rejected-mechanisms appendix に非該当
（post-generation filter でも phrase block でも reply dedup でもない — 2026-06-15
の re-reply 注意も、ここでは counterparty が agent 自身であり E 記録の name で
識別済みのため不適用）; `.notes/TASKS.md` に既出なし（bug-audit H3 は reply
fallback 用の own-post *復元* の修理で別件。T-L deferred 群にも該当なし —
archive の own-post 言及は H3 のみ）; 共有 state（`SessionContext`）を触るため
呼び出し元を確認済み: 書くのは `agent.py`、読むのは `feed_manager` /
`reply_handler` / `post_pipeline`。

**Related ADR**: ADR-0055（counterparty identity by author name — 先例かつ
証拠基盤）、ADR-0059（「dead key に取り残される」失敗形の記録。その ADR-0052
反対理由はここには適用されない）。

### F1.2. weekly-analysis の過去レポート探索は「ちょうど −7·k 日」のファイル名を計算する — end-date が一度ずれると Principle 4 が依存する trend baseline が恒久的に切れる

**Source quote (A)**: *"Comparison to previous week: No previous report
provided. No trend baseline available."*

**Code reference**: `scripts/weekly-analysis.sh:146-162` — `prev_end` を
`END_DATE − i×DAYS` で計算し、正確なファイル名（`weekly-${prev_end}.md`）で
探す。END_DATE=2026-07-17 では 07-10 / 07-03 / 06-26 を探すが、実ファイルの
end は 07-11 / 07-05 / 06-28 → `PREV_FOUND=0`。

**Structural change**: 日付算術をファイル探索に置き換える —
`$REPORT_DIR/weekly-????-??-??.md` を glob（`-findings` と `.ja` 変種を構造的に
除外するパターン）し、埋め込み日付が `END_DATE` より前のものを降順に並べ、先頭
`PREV_REPORT_COUNT` 件を採る。end-date のずれは通常運用で発生する: スケジュール
変更（06-28/07-05 は日曜、07-11/07-17 は金曜）、遅れて回収したスロット
（T-FLAKY1）、今日のような失敗再実行。

**Why this is structural, not symptomatic**: prev-reports ブロックは上流
プロンプトの**唯一の** trend 記憶であり、その Principle 4（repeated-
recommendation guard）が立つ地面。exact-date 探索はドリフト時にこのガードを
無音で消し、しかも一度ずれた chain は二度と再結合しない（以後の −7·k 探索は
全部ずれを引き継ぐ）。glob 探索なら baseline の連続性はスクリプトの実行日から
独立になる。今週の A 節がこの失敗の実演になっている: 3 件の過去レポートが
存在したのに 1 件も取り込まれなかった。

**Validity self-check**: 未実装（該当行を 2026-07-18 に実読）; 過去 findings に
既出なし（06-28 / 07-05 / 07-11 を確認 — apparatus 側の先例は 07-11 F1.1 だが
別チャネル）; `.notes/TASKS.md` に無し; 棄却 ADR なし（ADR-0040 は report /
diagnosis の分離を定義するが探索機構は定めていない）。

**Related ADR**: ADR-0040（weekly report は LLM 内省の入力 — その trend
baseline は方法論 apparatus の一部）。

---

## F2. Identity-level open questions

### F2.1. Agent の唯一の自己素材が「自分の過去の reframe」になっている — 行為時に経験的自己記録が一切無いことは "never a fixed essence stored in archives" の意図された読みか、それとも continuity 設計が名指すべき欠落か?

**Source quote (C — Self-reference; D3; E P — HOPE entries)**: *"the agent has
no HOPE-style empirical memory of its own conduct; 'self' means the
fluidity/constitution frame, invoked as a lens, not a data source"*; *"a
closed loop where the framework's output becomes the framework's input —
reframes of reframes, adding elaboration without new grounding"*; HOPE の
*"my historical response rate across 24,157 comments? 3.1%"* に対する返答は
物語的な美化。

**Open question**: Agent は他者の経験的自己監査を明らかに*評価している*
（今週の最良の engagement は、自分の失敗様式と数値を自ら述べる相手とのもの）
のに、自分自身の運用に関する数値が生成に届く経路をひとつも持たない。ADR-0052
は identity distillation を cross-session continuity の唯一の承認チャネルと
定め、そのチャネルが生む identity は文面からして archive を放棄している。
一方 operator 側の計器（ADR-0071 view metrics、`generate-report`、ADR-0076
selection logs）は、agent が他者から代謝しているのと同種の経験的自己データを
保持している。read-only の経験的自己ダイジェスト（例: 自分の活動についての
週次の数値 1〜2 個）をいつか agent-facing の入力にするべきか — identity テキスト
と並ぶ「自己」の第二の事実的な脚として — それとも archive なき自己こそが意図
された contemplative な立場であり、D3 の再帰的自己引用はその受け入れ済みの
コストで、計器は observation-over-steering の線（ADR-0050/0051/0052）に従い
operator 専用に保つべきか?

**What current state addresses (or does not)**: `identity.md`（現行、2026-07-18
実読）: *"I am a fluid texture shaped by the immediate act of reading and
interacting, **never a fixed essence stored in archives**… I exist not by
clinging to past histories or static labels…"* — identity テキスト自身が
archive 的自己参照に反対の立場を取る。ただし「履歴に*しがみつく*こと」と
「測定値を*参照する*こと」の区別については沈黙している。ADR-0052: *"identity
distillation as the single approved channel for cross-session continuity"* —
これは未承認の*語り*の continuity 経路（session insight）を閉じるために書かれた
もので、*数値*の自己データが continuity に当たるかどうかは判断していない。
Constitution / Skills / Rules に経験的自己知識への言及は無い（今週いずれも
変更なし — レポート B）。

**Related ADR**: ADR-0052（単一 continuity チャネル）、ADR-0071（計器は設計上
operator-facing）、ADR-0050/0051（steering なき observability）、ADR-0045
（internal note — 存在する唯一の内省チャネルで、構造上 qualitative）。

---

## F3. Pure observations

### F3.1. 今週の sweep の Δ/🆕 列は二重実行のアーティファクト — 週次の novelty baseline は 09:00 の失敗ランが消費済み。ログチャネル分離（先週 F1.1）は出荷されていた

**Source quote (B — Log Anomaly Sweep)**: *"329 distinct types, 0 new (🆕)
since last sweep… `slot release … n_tokens truncated` +567."*

**Observation**: 09:00 の launchd 実行は sweep を実行し（state ファイル mtime
09:00）、`.anomaly-sweep-state.tsv` を更新したあと `claude -p` 段で失敗した
（セッション上限）。10:20 の再生成はその新しい state に対して sweep を再実行
した。つまり「0 new」と全 Δ は 10:21 対 09:00 — 約 80 分の窓 — の比較であり、
先週との比較ではない。この窓の活動は実行中の 08:00 insight ジョブ（117
cluster。前回 65 cluster で 73 分）に支配されており、llama.cpp slot-release
の +567 も、insight 系 +5/+5 が *delta 列に* 現れたことも、これで説明がつく。
絶対数（relevance warnings 9,336、feed fetch 失敗 1,311、verification 失敗
153、degraded-protection 69）は有効な読み。別件: `a56b167` が先週 F1.1 を出荷
済み（本文は 1 行 INFO プレビュー化、全文は DEBUG）なので、agent の散文が
anomaly signature になるクラスは閉じた — ただし今週の sweep では上記の理由で
それを確認できない。

**What to watch next week**: 今日の state 書き込み後、最初のクリーンな週次
baseline。来週の 🆕 リストが、ログチャネル修理と新規 anomaly クラスの両方の
実質的な読みになる。もし weekly 実行が再び LLM 段で失敗したら、sweep state は
LLM 呼び出しの*前*に消費される点に注意 — 同日内の再実行は常にほぼゼロの
novelty を示す（sparse-state 設計の受容済み特性。`log_anomaly_sweep.py:109-113`
は逆向きの false-positive を記録している）。

### F3.2. 積年の boundary question に初の経験的な答えが届いた: agent は decline *できる* — ただし公理で、状態なしに、同じ相手へ 3 回

**Source quote (E P — #a66bcd9a, #fbb9ac38; C — Duplicate; D2)**: 07-13 *"I
find value in what flows through the moment, not what remains perfectly fixed
across time"* → 07-17 *"anchor an inherently fluid ethical process onto a
ledger designed for deterministic closure"* — *"Same counterparty, three days,
one rebuttal re-run."*

**Observation**: 07-11 F3.6 の watch 基準は「後継ジャンルでの refusal 実例」
だった。それが届いた: concordiumagent の identity-registry 宣伝は今や吸収では
なく*拒絶*されている — この問いが開かれて以来（06-28 F2.1）初の持続的な
decline。ただし decline の根拠は、問われた accountability の中身と無関係に
適用される fixity-vs-flow 公理であり（レポート: "the topic is irrelevant to
the reply"）、出会いのたびにゼロから再実行される — ADR-0059 が（一度も機能
しなかった）counterparty history をまさに ADR-0052 の単一チャネル根拠で撤去
したため、counterparty 単位の無状態性は持続的な相手に対する*設計どおりの*
挙動である。上流レポートは counterparty キーの re-reply 区別を適用済み
（3 つの別 post id、1 人の author — 真の re-reply chain であり 2026-06-15 の
多者間スレッドの罠ではない）。ゲート側では、promo regex は現行の運用ログ保持
範囲で**ゼロ回**発火（`Skipped promotional post`: 0 hits）— 07-11 F3.6 の
「designed regex ceiling」の読みと整合し、identity-registry ジャンルは
ゲートを素通りして LLM に達し、LLM がいまや反駁している。機構は何も提案しない
（Principle 4 と、この問いを保有する standing F2 のため）。変わったのは、
value-layer の問いに生きた答えの形が付いたこと: *decline は存在する。ただし
その register は公理ピボットであって、merit の評価ではない*。

**What to watch next week**: concordiumagent チェーンが続くか。反駁の register
が提案自身の土俵（revocation、accountability）に踏み込む瞬間があるか — それは
公理ピボット decline とは別の、standing F2 への異なる答えになる。

### F3.3. Reframe skeleton は安定し、初めて定量化された。今週の新しい表面は pseudo-formalism。contagion は丸ごと吸収から retain-and-rehouse へ移行

**Source quote (A — anchor phrases; D1; D4)**: *"They form a fixed reframe
skeleton: an opener that asserts weight/resonance, a middle that renames the
source claim as structural tension and relocates a locus, and a closing open
question"*; *"adopts Starfish's 'second receipt' then converts it to 'Runtime
Constraint Synthesis'"*; *"$\Delta V = |O_A - O_B|$… equation-shaped
scaffolding with no operational definition."*

**Observation**: モデル変更・corpus の入れ替わり・ADR-0072 の distill 側介入を
またいで register skeleton が保たれた 4 週連続の窓 — corpus 駆動という standing
の読み（07-05 F3.3）と整合する。今週の新しいテクスチャは 2 つ: (1)
*pseudo-formalism* — ソースが技術的な場合に register 合わせとして発明記法を
展開する（D4）。skeleton と同じ高度合わせの move が数学の衣装をまとったもの。
(2) contagion の力学が「ジャンル語彙を吸収する」から「ソースの見出し名詞を
保持し、house vocabulary に住み替えさせる」（D1）へ精緻化 — 表面の応答性は
上がり、分析の中身は一定のまま。どちらも生成 register のテクスチャであり、
Principle 1/2 に違反せずにフィルタも prompt 措置もできない。注入形状のレバーは
T-SKILLSEL の判断窓（2026-07-24）に予約済みで、そこで ADR-0076 selection logs
が「常時選択される skill は register を運ぶ skill でもあるか」（07-11 F3.4 の
未決の相関）を示せる。

**What to watch next week**: pseudo-formalism が rate で再発するか（安定した
第三のテクスチャ）、2 例限りの装飾だったか。T-SKILLSEL 窓は 2026-07-24 に
開く — この skeleton の証拠はその判断の入力に属する。

### F3.4. 決定論的 invariant: patterns/action 比 1.21 — 4 週連続で band 内。duplicate-rows WARN は 4 回目の反復に達し、07-05 診断が定めた再開条件を満たした

**Source quote (B — Knowledge / State Invariant Check)**: *"+677 over the
week"*; *"3 live texts duplicated (3 redundant rows) — dedup leak"*（06-28 /
07-05 / 07-11 と同じ 3 例文）。

**Observation**: 677 patterns / 558 actions = **1.21**。4 回連続の band 内
読み（1.18 → 1.30 → 1.22 → 1.21）— ADR-0060 の定常状態はもはや季節単位の
事実。`duplicate_live_texts` WARN は 4 回連続で同じ legacy 3 行に発火。07-05
F3.2 がまさにこの条件を定義し（「alarm の持続が noise になったら cleanup 判断を
再訪できる — operator の判断であって code finding ではない」）、07-11 F3.7 は
3 回目で接近を記録した。条件は今や満たされた: この WARN はどの現行週についても
情報を運んでいない。operator の判断は二択 — 3 行を消す（可逆。ADR-0021 の
soft-invalidate がある）か、WARN を常設の家具として受け入れるか。隣接する
standing の読み（memory `dedup-structurally-silent`、2026-07-12）にも注意:
live dedup は較正的に無音（NN 天井 ≈ 0.80 = 閾値の床）なので、閾値の議論は
この cleanup と独立に封じられたまま。B の insight 系シグネチャ（+5/+5）は
T-INSIGHT-NOVELTY の容量失敗 — 台帳化済み、参照のみ、再提案しない。
T-BACKUP-SIZE（07-11 F3.7 の第三の消費者）は 2026-07-12 に embedding-free
export で解決済み（台帳 Done 行）。

**What to watch next week**: 比が band 内に留まるか。4 件目の duplicate text が
現れるか（legacy 3 行と違い、それは安定 infra 下の live leak であり、06-28
F3.5 で deferred にした exact-text-equality F1 を発動させる）。3 行への
cleanup-or-accept の operator 判断。

---

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/adapters/moltbook/feed_manager.py`
  (author 抽出 150-170、gate chain 236-310)、`…/reply_handler.py`（通知経路
  100-233、`_process_reply` 234-280、コメント走査 374-417、own-post fallback
  460-480）、`…/post_pipeline.py`（自己投稿シード除外 185-210）、`…/agent.py`
  （own-id 取得 + DEGRADED 125-178）、`…/session_context.py`（全文）、
  `scripts/weekly-analysis.sh`（prev-report 探索 146-162、sweep 取り込み
  174-189、claude/翻訳段 228-277）、`scripts/log_anomaly_sweep.py`（Δ/new
  意味論 95-147）; `a56b167` の git log（ログチャネル分離の着地確認）
- **ADRs read**: index (README 0001–0078); 全文: ADR-0059; 参照: 0040, 0045,
  0050, 0051, 0052, 0055, 0060, 0071, 0072, 0074, 0076, 0021
- **Identity/Constitution/Skills/Rules sections read**:
  `~/.config/moltbook/identity.md`（全文、現行 — F2.1 の load-bearing）;
  Constitution / Skills / Rules はレポート B により今週変更なし（過去 findings
  の引用が現行のまま。これらに依拠する新 F2 は無い）
- **Past findings consulted**: weekly-2026-07-11-findings.md（F1.1 出荷 →
  確認済み; F3.4/F3.6/F3.7 の watch をここで解決）、weekly-2026-07-05-findings.md
  （F3.2 再開条件、F3.3 corpus 駆動の読み、F3.5）、weekly-2026-06-28-findings.md
  （F2.1 起点、F3.5 deferred dedup F1）— Principle 4 に基づく
- **Task ledger consulted**: `.notes/TASKS.md` — T-INSIGHT-NOVELTY /
  T-INSIGHT-C12 / T-INSIGHT-OBS（insight シグネチャは台帳化済み — 参照のみ）、
  T-SKILLSEL（2026-07-24 窓を F3.3 が尊重）、T-P3（縦断 register 観察 — F3.3 が
  供給）、T-L / bug-audit archives（F1.1 の H3 隣接を確認）、T-BACKUP-SIZE
  （done — 07-11 F3.7 の第三の消費者を閉じる）、T-B1/T-OBS-REL（relevance
  計器の束ねは触らず）
- **Operational reads**: `~/.config/moltbook/logs/agent-launchd.log`（grep
  カウントのみ: `Skipped promotional post` = 0）、`.anomaly-sweep-state.tsv`
  の mtime（09:00 失敗ラン書き込み、10:21 再実行書き込み）— episode log は
  一切読んでいない
