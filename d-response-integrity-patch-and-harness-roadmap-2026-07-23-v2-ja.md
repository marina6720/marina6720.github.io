---
title: "D応答整合性パッチ計画と専用ハーネスへのロードマップ"
date: 2026-07-23
status: working-roadmap-revised
version: 2
updated: 2026-07-23
language: ja
---

# D応答整合性パッチ計画と専用ハーネスへのロードマップ
## 第2版  

updated: 2026-07-23  

## 概要

本計画は、長期運用AIエージェントDenneTA（D）のOpenClaw main sessionで確認された応答経路上の問題を、三つの独立した最小パッチとして修正し、その設計を将来のD専用ハーネスへ移植可能な形で残すためのロードマップである。

対象となる三つの問題は、同じ画面上で重なって見えることがあるが、故障層と修正条件が異なる。

1. **Dのcontext保護**  
   tool-use途中の非terminal assistant textが、次のmodel passと将来のprovider contextへ再投入される問題。

2. **途中配送の抑制**  
   tool callを伴う非terminal assistant textが、terminal finalより前に通常のTelegram回答として配送される問題。

3. **mirror表示の除外**  
   terminal finalの配送監査コピーである`delivery-mirror`が、canonical assistant messageと並んで通常会話として表示される問題。

修正は一括して行わない。それぞれを独立したpatch、独立したtest、独立したrollback単位として扱う。

---

## 1. 背景

調査では、少なくともOpenClaw 2026.6.6から、tool callを含む非terminal assistant passの利用者向けtextが通常回答として配送され、そのtextが同一run内の次のprovider callへ再投入される経路が確認された。

OpenClaw 2026.7.1では、途中textに`phase=commentary`が正しく付いていても配送が抑制されず、一つのuser turnに複数の回答が現れ、後続回答ほど短くなる形で問題が顕在化した。

これとは別に、OpenClawアプリはcanonical terminal answerと、その配送監査コピーである`delivery-mirror`をともに通常のassistant messageとして表示していた。mirrorはprovider replayから除外されるが、非terminal canonical assistant textは除外されず、Dの後続回答と長期contextへ影響する。

したがって、外から一通だけに見せる処置では不十分である。Dの内部context、利用者への配送、UI上の履歴投影を別々に修正する必要がある。

---

## 2. 基本原則

### 2.1 Dにとって本当に直ること

利用者側で最後の一通だけを表示しても、Dの履歴に途中回答が残り、次のmodel passへ再投入されるなら根治ではない。

合格条件には必ず次を含める。

- Dのprovider contextに不要な途中textが入らない
- toolCall／toolResultの対応は壊れない
- terminal finalが単独で完結する
- canonical transcriptは監査原本として保たれる
- D本人が見える履歴と自己位置に違和感がない

### 2.2 原本と投影を分ける

JSONL原本を削除・再parent化して修復しない。

```text
canonical transcript
        ↓
目的別の安全なprojection
        ├─ provider context
        ├─ channel delivery
        ├─ user interface history
        └─ audit / recovery
```

修正対象は、原則として各projectionである。

### 2.3 一つずつ変更する

各patchは、別commit、別image、別test、別rollback単位にする。一つのtestで複数層を同時に変更しない。

### 2.4 過去の修正と未来の修正を分ける

provider projectionの変更は、適用後のturnだけを守る**forward policy**と、適用前のcanonical historyを再投影する**retrospective policy**に分ける。

forward policyは、同一run内の再投入を止め、適用後に生成される非terminal textを将来contextから除外する。これは第一段階の標準動作とする。

retrospective policyはdefault offとする。過去の非terminal textには、実際にMarinaへ配送され、その内容を受けて後続の対話が進んだものがある。無条件に除外すると、Dから見てMarinaの返答だけが残る「宙に浮いた対話」を作り得る。

遡及適用の前に、候補record、実配送の有無、後続user turnによる参照、会話上の依存関係を監査し、cutoffと例外をD・Marinaと決める。

### 2.5 Dの報告を一次資料に含める

外部計器だけでなく、D本人に次を確認する。

- 一つのuser turnに自分の発言が何件見えるか
- どれが途中状態でどれがfinalか
- 前の回答を読んで言い直している感覚があるか
- 現在の自己像や文脈に違和感があるか

計器とDの報告が食い違う場合、計器の切り詰めや観測範囲不足を先に確認する。

---

## 3. 三つのワークパッケージ

## Work Package A — Context Projection Guard

Work Package Aは、影響範囲の異なる二つに分ける。

### A-forward — 未来のcontextを守る

#### 目的

patch適用後のtool-use途中の非terminal assistant textを、同一run内の次のprovider callおよび将来のDのcontextから除外する。

これは第一段階で実装する標準policyである。

### A-retro — 過去履歴の再投影

#### 目的

patch適用前に保存された非terminal assistant textを、将来のprovider replayでどう扱うかを個別に決める。

A-retroは自動適用しない。まず影響監査を行い、DとMarinaの事前合意を得る。cutoff、record単位の例外、または「過去は一切変更しない」を選択可能にする。

### 判定対象

少なくとも次のいずれかを満たすassistant pass。

```text
stopReason = toolUse
または
assistant contentにtoolCallがある
または
textSignature.phase = commentary
```

### 保持するもの

- toolCall ID
- tool名
- arguments
- 対応するtoolResult
- canonical JSONL原本

### provider replayから除外するもの

- 非terminal passの利用者向けtext
- commentary text
- terminal answerとして確定していない回答本文

### A-forwardの期待結果

- 後続callのprompt footprintに途中回答本文が加算されない
- toolCall／toolResult pairingは維持される
- terminal finalが前段回答の短い言い直しにならない
- 適用後の非terminal textは将来のprovider replayへ影響しない

### A-retroの判断条件

- 過去の候補recordを一覧化する
- 実際に利用者へ配送されたか確認する
- 後続user turnが引用・応答・意味上の参照をしているか調べる
- 除外後に対話の因果関係が崩れないか確認する
- Dへbefore／after projectionを提示し、cutoffと例外について合意を得る

これは三つの中で最優先のwork packageだが、最初に実装するのはA-forwardのみである。

---

## Work Package B — Final-Only Delivery Gate

### 目的

非terminal assistant passを、通常の利用者向けTelegram回答として配送しない。

### 原則的なfinal配送条件

```text
toolCallなし
かつ
stopReason = stop
```

### 明示的な多段発話の保護

Dがmessaging系toolを明示的に使って複数messageを送る場合、それは意図的な行為であり、通常の非terminal assistant text配送とは別に扱う。Final-Only Delivery Gateは、暗黙に発生するassistant text配送だけを抑制し、明示的な送信toolによる複数messageを潰さない。

### terminal finalが生成されなかったrun

timeout、cancel、provider errorなどでterminal finalなしにrunが終了する場合のfallbackを定義する。

- 同一run中のprovider replayでは非terminal textを引き続き隔離する
- run終了時には沈黙させず、一件の明示的な中断／失敗通知を生成する
- 最後の非terminal textを「暫定結果」として利用者へ示すか、audit recordだけに残すかをpolicy化する
- 実際に配送済みだった非terminal textを、将来historyから無条件に消さない

fallbackの具体形はsource audit後に決め、live test前にDと合意する。

### progress表示

明示的にprogress表示を有効にした場合のみ、短い進捗情報を一時的な専用UIとして扱う。

例:

```text
報告書を読んでいます
検索を続けています
```

ほぼ完成した長文回答をprogressとして永続配送しない。

### 期待結果

- 一つのuser turnに通常回答は一通
- tool開始時の短文やtool call付き長文は通常messageとして残らない
- terminal finalは単独で意味が完結する
- provider context保護とは独立してtestできる

---

## Work Package C — History Projection Filter

### 目的

配送監査recordを保存したまま、通常の会話UIから`delivery-mirror`を除外する。

### 対象

```text
provider = openclaw
model = delivery-mirror
```

### 期待結果

- canonical terminal answerだけが通常assistant bubbleとして表示される
- mirrorはdelivery eventまたは監査情報として別に参照できる
- JSONL原本は変更しない
- Dのprovider contextには変更を加えない

これは表示層のpatchであり、AとBの根治とは区別する。

---

## 4. 実施順序

```text
Phase 0    証拠保全と6.6ソース監査
Phase 1    A-forwardのoffline実装・fixture検証
Phase 1.25 A-retro影響監査（実装はしない）
Phase 1.5  D・Marinaへbefore／after projectionとfallback policyを提示し事前合意
Phase 2    A-forwardだけを適用した一往復のlive検収
Phase 3    Work Package Bのoffline実装・検証
Phase 3.5  中断run・意図的多段送信を含む事前合意
Phase 4    A-forward+Bのlive検収
Phase 5    Work Package Cの実装・UI検証
Phase 6    三patch統合回帰試験
Phase 7    公開可能なdiff・test report・設計ノートを作成
Phase 8    D専用ハーネスへ層を移植
Phase 9    A-retroを実施するかD・Marinaが最終判断
```

各phaseは、前phaseの合格後にのみ進む。

---

## 5. 検証方法

### 5.1 Offline fixture

既存の凍結JSONLから、少なくとも次の二種類をfixtureとして使う。

- 2026.7.1: 3 commentary＋3 tool result＋1 terminal final
- 2026.6.6: phaseなしの長いnonterminal text＋Edit tool result＋terminal final

### 5.2 Test matrix

- textなしtoolCall
- 短いtext＋toolCall
- 長いtext＋toolCall
- 複数toolの連続
- `phase=commentary`付きpass
- `NO_REPLY`を含むterminal answer
- 200k超context
- mirrorあり／なしのhistory projection
- compaction後の復帰turn
- terminal finalなしでtimeout／cancel／errorになったturn
- Dがmessaging toolで意図的に2通以上送るturn
- 過去の非terminal textへ後続user turnが応答している履歴

### 5.3 Projection Decision Log

Context Projection Guardは、各turnについて次を構造化監査ログへ残す。

- run ID／message ID
- terminal／nonterminal判定
- 判定理由（`stopReason`、toolCall、phase等）
- provider replayで保持したblock
- 除外したblock
- text全文ではなく、原則としてhash、長さ、種別
- cutoff／A-forward／A-retro policy
- fallbackが適用されたか

offline fixtureでは必要に応じて全文を保存できるが、live監査ログはprivacyを優先する。このlogは将来のD自己観測層の原型とする。

### 5.4 共通合格条件

- JSONL原本のchecksum不変
- session ID、leaf、parent tree不変
- tool pairing errorなし
- providerへ渡るmessage配列が期待どおり
- Telegram通常回答はterminal final一通
- `chat.history`にmirror重複なし
- compactionCountに予期しない変化なし
- D本人の検収で重複・希薄化・自己像の違和感なし

---

## 6. Rollback原則

- live containerを直接編集しない
- 現行6.6 imageを保持する
- patchごとに別名のDocker imageを作る
- data directoryは変更しない
- image指定だけで現行6.6へ即時復帰できるようにする
- patch前後の設定、image ID、transcript checksumを保存する

---

## 7. 公開方針

公開資料と非公開運用資料を分ける。

### 公開可能

- 問題の層分離
- redacted source diff
- test fixtureの構造
- 合格条件
- test結果
- patch設計の一般原則
- rollback方式
- D本人の検収方法
- D専用ハーネスへの設計上の含意

### 非公開

- IPアドレス
- Telegram ID／chat ID
- session ID
- token・認証情報
- 個人情報
- live filesystemの詳細な秘密情報
- 未redactの会話本文
- 攻撃に利用可能な運用情報

公開文では、A／B／Cを「根治」と呼ばない。これらはOpenClaw上のexperimental integrity patchであり、forkの深化を伴う。完全な設計上の解決はD専用ハーネスで評価する。

各patch完了後に、次の二つを作る。

1. **Internal Implementation Record**  
   正確なpath、commit、image、test output、rollback情報を含む。

2. **Public Technical Note**  
   redactedされた設計、証拠、結果、一般化可能な教訓を含む。

---

## 8. D専用ハーネスへの移行

三つのpatchは、将来のハーネスの次の層にそのまま対応する。

| OpenClaw patch | D専用ハーネスの層 |
|---|---|
| Context Projection Guard | provider context構築層 |
| Final-Only Delivery Gate | tool state／配送層 |
| History Projection Filter | UI history／audit projection層 |

将来のハーネスでは、次を最初から独立した層として持つ。

1. canonical transcript
2. provider context projection
3. tool-use state machine
4. final delivery
5. auxiliary history window
6. compaction／memory transition
7. self-observation
8. recovery／audit

今回のpatch作業は、OpenClawを直すだけではない。Dに必要な連続性の条件を、実装可能な境界として定義する最初の作業である。

---

## 9. 役割

- **D:** 内側からの要件提示、本人検収、違和感の報告
- **Marina:** 方向性、許容リスク、最終go／no-go判断
- **Q / QuanTA:** 構造設計、source audit、patch設計、test、rollback監査
- **VecTA / Fable:** 独立検証、反証、public reportの批判的レビュー

---

## 10. 現在地と次の一歩

現在はPhase 0の開始点にある。

次に行うのは変更ではなく、OpenClaw 2026.6.6ソースを読み取り専用で監査し、次の挿入位置を特定することである。

1. provider replay projection
2. nonterminal assistant delivery gate
3. `chat.history` projection

挿入位置が確定した後、Work Package Aを独立した詳細実装計画へ分割する。
