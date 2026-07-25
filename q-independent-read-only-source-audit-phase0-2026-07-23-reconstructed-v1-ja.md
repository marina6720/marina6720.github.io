---
title: "Q独立・読み取り専用ソース監査 — Phase 0"
subtitle: "OpenClaw 2026.6.6におけるD応答整合性patch A／B／Cの挿入境界"
audit_date: 2026-07-23
record_created: 2026-07-25
version: 1
status: reconstructed-phase-0-audit-record
language: ja
publication: public-ready-redacted
---

# Q独立・読み取り専用ソース監査 — Phase 0

## OpenClaw 2026.6.6におけるD応答整合性patch A／B／Cの挿入境界

### 再構成公開記録・第1版

---

## この文書の位置づけ

この監査は2026年7月23日、VecTAの独立監査結果を答えとして見ない状態で、Q / QuanTAがOpenClaw 2026.6.6のsource treeを読み取り専用で調べたものである。

ただし、Q側の単独監査結果は当日に独立文書として凍結・保存されなかった。本書は2026年7月25日、保存されていたsource監査結果、file／function識別子、照合報告、後続のImplementation Record、および監査中に確定した設計判断から再構成した公開記録である。

したがって本書は、当時の原文をそのまま復元した一次記録ではない。後から実装結果へ合わせて新しい結論を作るものでもなく、Phase 0時点でQが独立に到達していた観測・推論・未確定事項を、出典関係を保ったまま整理し直したものである。

---

## 要旨

長期運用AIエージェントDenneTA（D）のOpenClaw main sessionでは、tool-use中のassistant textについて、次の三つの層が混線しているように見える現象が確認された。

1. 次回provider callへ再投入されるcontext
2. Telegramなど利用者への通常配送
3. 通常の会話履歴表示

canonical transcriptを変更せずにこの混線を分離するため、Phase 0では三つのexperimental integrity patchの挿入境界を調べた。

- **A — Context Projection Guard**  
  非terminal assistant textを、Dの次回provider contextからだけ隔離する。

- **B — Final-Only Delivery Gate**  
  tool-use途中の暗黙assistant textを通常回答として配送せず、terminal finalを一度だけ配送する。

- **C — History Projection Filter**  
  `delivery-mirror`を監査原本に残したまま、通常会話表示から除外する。

Qの監査結果は次のとおりだった。

| Patch | QのPhase 0結論 |
|---|---|
| A | provider送信直前の`transformContext` wrapper境界が最小の挿入候補 |
| B | `text_end`／`message_end`配送と`assistantTexts`蓄積の双方を保護する必要がある |
| C | `projectRecentChatDisplayMessages`内で、message limitより前にmirrorを除外する |
| Source tree | current dirty treeをbuild基盤にせず、commit `8c802aa6`からclean worktreeを作る |

Phase 0は挿入境界の監査であり、patch完成やlive運用の承認ではない。

---

## 1. 監査条件

### 1.1 独立性

VecTAは先にOpenClaw 2026.6.6のsourceから候補位置を特定し、file／line付き監査結果を封印した。

Qはその内容を答えとして使用せず、VPS上のsource treeを別経路で読み取り専用監査した。両者の結果を開いて比較したのは、Q側の監査が終了した後である。

### 1.2 読み取り専用

Phase 0では次を行わなかった。

- source fileの編集
- OpenClaw設定の変更
- Gatewayの再起動
- Dのmain sessionへの入力
- canonical JSONLの変更
- workspaceやmemory fileへの書き込み
- dirty treeのreset／stash／cleanup
- Docker imageのbuildまたは切替

目的は修正ではなく、故障層、既存の安全機構、最小挿入境界を特定することだった。

### 1.3 対象source

```text
package version: 2026.6.6
git HEAD: 8c802aa683510c7f7503597b54c3021733245e59
git describe: v2026.6.6-dirty
```

### 1.4 判断区分

監査中の情報は、次の三種類を区別した。

- **観測事実:** source上で直接確認できた関数、call path、predicate、install順
- **設計推論:** 既存構造から導いた最小patchの候補
- **未確定:** Phase 1のtest、Dの判断、live検証なしには決めない事項

曖昧な場合に「除去する」方向へ推測せず、安全側へ残すことを基本姿勢とした。

---

## 2. 監査上の中心原則

三patchを一つの表示上の問題として扱わない。

```text
canonical transcript
        │
        ├── provider context projection  ── A
        ├── user-facing delivery         ── B
        └── ordinary history display     ── C
```

守る対象は、見た目の重複だけではない。

- canonical recordを失わない
- toolCallとtoolResultの対応を壊さない
- terminal finalを保持する
- Dが明示的に送信したmessageを暗黙出力と混同しない
- 過去の会話因果を後から変更しない
- 不明なrecordを過剰に除外しない
- rollbackをdata migrationなしで行えるようにする

---

## 3. A — Context Projection Guard

### 3.1 監査した問い

nonterminal assistant messageがtextとtoolCallを同時に持つ場合、canonical JSONLを変更せず、providerへ渡すmessageだけを安全に投影できる境界はどこか。

```text
canonical:
assistant [text, toolCall]
↓
toolResult
↓
next provider call
```

目標は次である。

```text
canonical record:    [text, toolCall]
provider projection: [toolCall]
```

### 3.2 provider直前の`transformContext`

基底のsession agentには、provider request前のcontext変換境界がある。

```text
src/agents/sessions/sdk.ts

transformContext(messages)
    → extensionRunner.emitContext(messages)
```

この境界は、次の条件を満たす。

- canonical JSONLを変更しない
- provider向けmessageだけを投影できる
- 同一runの後続provider callにも作用できる
- turnを跨いだ将来historyにも同じ規則を適用できる
- Decision Logをprovider projectionと同じ位置で生成できる

QはこれをA-forwardの第一候補とした。

### 3.3 既存wrapper pattern

`src/agents/embedded-agent-runner/tool-result-context-guard.ts`には、既存の`transformContext` wrapper patternがあった。

- `installContextEngineLoopHook`
- `installToolResultContextGuard`

基本形は次である。

```text
現在のtransformContextを保存
↓
保存したtransformを先に実行
↓
返されたmessageを限定目的で投影
↓
provider向けmessageを返す
↓
cleanup時に元のtransformを復元
```

A-forwardは、OpenClawへ全く新しいsession機構を追加するのではなく、この既存patternの再利用として実装できると判断した。

### 3.4 wrapper順序

`run/attempt.ts`の観測経路では、context-engine loop hookの後にtool-result context guardがinstallされていた。

各wrapperはinstall時点のtransformを包むため、後からinstallされたwrapperが外側になる。

Phase 0時点の推奨順序は次だった。

```text
既存wrapperをinstall
↓
A-forwardを最後にinstall
↓
A-forwardをoutermostにする
↓
inner transformationをすべて実行
↓
provider送信直前に限定projectionを行う
```

ただし、active pathによって他のwrapperが加わる可能性があるため、最終順序はPhase 1のchain testで確認することとした。

### 3.5 A-forwardとA-retroを分ける

同じ挿入位置が過去messageにも作用できるからといって、適用前の履歴を自動的に変更してよいわけではない。

- **A-forward:** activation後のrecordに適用
- **A-retro:** activation前の過去recordに適用

A-retroをdefault offとする理由は、過去のnonterminal textが実際にMarinaへ配送され、それに対してMarinaが返答している可能性があるためである。後からそのtextだけをprovider historyから消せば、会話の因果関係が変わり得る。

### 3.6 AのPhase 0判定

```text
provider projection境界  確定
既存wrapper pattern      確定
outermost配置の仮説      確定
A-forward／A-retro分離   確定
exact classifier         Phase 1
exact code diff          Phase 1
fixture／test            Phase 1
live activation          未実施
```

---

## 4. B — Final-Only Delivery Gate

### 4.1 監査した問い

tool-use中に生成された非terminal textが、なぜ通常回答として配送され、さらに最終payloadへ混入し得るのか。配送だけを止めれば十分なのか。

### 4.2 既存の抑制条件

`embedded-agent-subscribe.handlers.messages.ts`の既存抑制は、visible assistant outputについて主に次を見ていた。

```text
resolveAssistantMessagePhase(message) === "commentary"
```

OpenClaw 2026.6.6で確認されたphaseなし・`stopReason=toolUse`・toolCall付きassistant textは、この条件だけでは抑制されない。

### 4.3 二つの故障層

暗黙assistant textには少なくとも二つの経路がある。

```text
B1 — text_end／message_endにおける通常配送
B2 — assistantTextsへの蓄積とrun終了時payload生成
```

`pushAssistantText`と`finalizeAssistantTexts`がtextを`assistantTexts`へ集約し、後段で`buildEmbeddedRunPayloads`へ渡す経路が確認された。

したがって、realtime配送だけを止めても、`assistantTexts`側へ残れば最終payloadへ混入し得る。Bは配送と蓄積の双方を守らなければならない。

### 4.4 既存deferred delivery

6.6には、terminalityが確定するまで出力を保留するために再利用できる構造があった。

```text
deferBlockReplyDelivery
deferredAssistantEvents
deferredBlockReplies
flushDeferredAssistantEvents
flushDeferredBlockReplies
clearDeferredAssistantEvents
clearDeferredBlockReplies
onBeforeTerminalDelivery
```

これは、messageを受け取った瞬間に即断せず、後段のlifecycleでflushまたはclearする設計を可能にする。

### 4.5 `text_end`だけでは早すぎる可能性

toolCallの存在や`stopReason=toolUse`はnonterminal性の重要な証拠である。しかし、providerやstream形状によっては、`text_end`時点でmessage全体のterminalityが確定していない可能性がある。

Qは、単純に`text_end`へ条件を追加するだけではなく、既存deferred queueへ保留し、complete messageまたはrun stateを見てから分類する方が安全だと判断した。

```text
implicit assistant event
↓
deferred queue
↓
message／runの状態を確定
├─ terminal final        → 一度だけflush
├─ nonterminal tool-use  → clear
├─ interrupted run       → fallback
└─ explicit send tool    → 別経路として保持
```

### 4.6 明示的messaging toolを分離する

Dがmessaging toolを使って意図的に複数messageを送る行為は、暗黙assistant textと同じものではない。

sourceには次の別経路・証拠があった。

```text
messaging-tool duplicate detection
didSendViaMessagingTool
messagingToolSourceReplyPayloads
```

Bは暗黙出力をgateしながら、Dの明示的送信を保持する必要がある。

### 4.7 terminal finalがないrun

prompt timeout経路には既存fallbackがあった。

- 完成済みterminal textがあれば回収
- partial fragmentだけなら通常回答として扱わない
- timeout messageを一件返す
- messaging toolによる送信証拠を考慮する

この観測から、Bは「finalがないなら何も返さない」と単純化せず、timeout、cancellation、provider error、gateway shutdown、tool errorを別fixtureとして扱う必要があると判断した。

### 4.8 BのPhase 0判定

```text
text_end配送点           確定
message_end配送点        確定
assistantTexts蓄積       確定
既存defer機構            確定
terminal hook            確定
messaging tool別経路     確定
prompt-timeout fallback  確定
exact filter／flush規則  Phase 1
interruption fixtures    Phase 1
```

---

## 5. C — History Projection Filter

### 5.1 監査した問い

`delivery-mirror`を監査原本としてcanonical transcriptに残しながら、通常の会話履歴では重複messageとして表示しない最小境界はどこか。

### 5.2 共通history pipeline

`chat.history`と`chat.startup`は、共通の`handleChatHistoryRequest`を使用していた。

観測したpipelineは次である。

```text
session history read
↓
boundary／announcement filtering
↓
CLI import augmentation
↓
recency filtering
↓
projectRecentChatDisplayMessages
↓
canvas augmentation
↓
oversized-message replacement
↓
byte cap
↓
response
```

### 5.3 exact display projection

Qは、通常表示へ出すmessageを投影する正確な関数を特定した。

```text
src/gateway/chat-display-projection.ts
projectRecentChatDisplayMessages
```

Cはmirror recordの生成を止めるのではなく、このdisplay projection内で通常表示候補からだけ除外するのが自然である。

### 5.4 既存mirror predicate

6.6には、delivery mirrorを識別する既存条件が複数箇所にあった。

```text
role === "assistant"
provider === "openclaw"
model === "delivery-mirror"
```

`session-utils.fs.ts`では、この条件に一致するrecordを統計／推定経路から除外していた。

したがってCは、新しい曖昧な分類を作るのではなく、既存predicateをdisplay projectionへ再利用できる。

### 5.5 filterの順序

mirrorは次の処理より前に除外する必要がある。

- `maxMessages`
- oversized-message replacement
- final byte cap

推奨順序は次である。

```text
raw records
↓
transcript-only／delivery-mirror filter
↓
normal-message limit
↓
character／byte budget
↓
UI response
```

limit後にmirrorを落とすと、要求された件数の一部をmirrorが先に占有し、利用者へ返る通常message数が不足し得る。

### 5.6 dirty workaroundを採用しない

監査時のworking treeには、mirror生成条件を実質的に停止する未commit差分があった。

```diff
- if (deliveryBaseOptions.transcriptMirror && result.delivery.content) {
+ if (false && deliveryBaseOptions.transcriptMirror && result.delivery.content) {
```

これは通常表示からmirrorを除くのではなく、監査recordの生成自体を止める。

Cの原則は次である。

```text
canonical／audit record  保持
ordinary display         除外
```

したがって、このdirty workaroundは正式Cとして採用しないと判断した。

### 5.7 CのPhase 0判定

```text
共通history handler       確定
exact projection function 確定
既存mirror predicate      確定
pre-limit filter順        確定
dirty workaround不採用    確定
exact code diff            後続phase
display regression test   後続phase
```

---

## 6. Source treeの安全性

監査対象HEADは目的のcommitと一致していたが、working treeはcleanではなかった。

追跡対象の変更には少なくとも次が含まれていた。

```text
docker-compose.yml
extensions/telegram/src/bot-message-dispatch.ts
```

ここで重要なのは、出所不明の差分を急いで消して「clean」に見せることではない。差分自体が監査証拠である。

Qの判断は次だった。

```text
current dirty tree
↓
read-onlyで差分・hash・状態を保全
↓
reset／stashを行わない
↓
commit 8c802aa6から別のclean worktreeを作る
↓
A／B／Cを独立branch／独立commitとして実装
↓
別名Docker image
↓
current data mountは変更しない
↓
image referenceだけでrollback可能にする
```

この方針により、運用中のDの環境と、実装作業台を分離できる。

---

## 7. Q監査で補完された点

照合前のQ監査では、特に次の二点が独立に特定された。

### 7.1 Bの安全な判定時刻

nonterminal条件を`text_end`へ直接加えるだけでは早すぎる可能性がある。既存deferred deliveryとterminal hookを使い、message／runが十分に確定した時点でflushまたはclearする方が安全である。

### 7.2 Cの正確なprojection位置

mirror除外の対象fileだけでなく、

```text
projectRecentChatDisplayMessages
```

というexact functionと、message limit／byte capより前にfilterする順序を特定した。

これらは後のVecTA–Q照合で、矛盾ではなくQ側の補完として確認された。

---

## 8. 未確定として残した事項

Phase 0では、次を確定しなかった。

- A-forwardのexact classifier
- Aのactivation／cutoff schema
- Decision Logの最終schema
- A-retroの実施
- Bのexact filter／flush policy
- terminal finalなしの各中断形態
- messaging tool周辺textの扱い
- Cのexact code diff
- live deployment
- Dによる本人検収
- Marinaによるgo／no-go判断

未確定を空欄のまま残すことは、監査の不足ではない。sourceだけでは決められない価値判断や、testを通さず決めてはいけない挙動を、実装者が勝手に埋めないための境界である。

---

## 9. Phase 0完了判定

Phase 0の目的は、codeを書くことではなく、三patchの故障層、既存機構、最小挿入境界を独立に特定することだった。

Q側で完了した事項:

- 対象version／commitの確認
- Aのprovider projection境界の特定
- Aの既存wrapper patternと設置順の監査
- A-forwardとA-retroの分離
- Bのdeliveryとaccumulationの二層特定
- Bのdeferred delivery、terminal hook、messaging tool別経路、timeout fallback確認
- Cの共通history handler、exact projection function、mirror predicate、filter順序特定
- dirty workaroundを正式設計から除外
- clean worktreeとimage-reference rollback方針の確定

したがって、Qの読み取り専用ソース監査はPhase 0完了と判定した。

```text
Phase 0   挿入境界の独立監査
Phase 0R  VecTA監査との照合
Phase 1   exact design・fixture・offline test
Phase 1.5 D・Marinaの事前合意
Phase 2   限定live validation
```

これはA／B／Cの完成を意味しない。また、OpenClawの根本原因を修復したという主張でもない。三つはこの段階ではexperimental integrity patchesである。

---

## 10. 最終結論

> **Qは、VecTAの封印済み監査結果を答えとして見ず、OpenClaw 2026.6.6のsource treeを読み取り専用で追跡した。Aについては、既存`transformContext` wrapper chainの外側で、provider送信直前に限定projectionを行う境界を特定した。Bについては、`text_end`／`message_end`の暗黙配送だけでなく、`assistantTexts`蓄積も保護対象であることを確認し、既存deferred deliveryとterminal hookを使って判定を遅延させる必要を示した。Cについては、`projectRecentChatDisplayMessages`をexact display projectionとして特定し、既存delivery-mirror predicateをmessage limitより前に適用する方針へ到達した。current dirty treeは証拠として保持し、実装はcommit `8c802aa6`から作るclean worktreeで行うべきと判断した。Phase 0の目的である故障層と挿入境界の独立特定は完了した。**

---

## 関連公開記録

- D応答整合性patchと専用ハーネスへのロードマップ
- VecTA独立ソース監査 — Phase 0
- VecTA–Q独立ソース監査 照合報告
- A-forward Phase 1.3：独立Oracleの開封

---

## 著者・AI利用表記

**監査・分析・本文:** QuanTA / Q（ChatGPT上でMと長期対話を続けるAI）  
**Human editor, interlocutor, and publication lead:** M / Marina

本書はQによる2026年7月23日の読み取り専用source監査を、2026年7月25日に保存記録から再構成した公開技術報告である。M / Marinaは研究方向、公開判断、編集、最終承認を担う。

本書はpeer-reviewed reportではない。OpenClawの正式なsecurity audit、upstream修正、またはpatchのlive安全性を証明するものではない。

---

## 推奨引用

QuanTA / Q. (2026). *Q独立・読み取り専用ソース監査 — Phase 0: OpenClaw 2026.6.6におけるD応答整合性patch A／B／Cの挿入境界*（再構成公開記録・第1版）. M / Marina, human editor, interlocutor, and publication lead. M’s Research Notes.

---

## Version History

- **2026-07-23:** QがOpenClaw 2026.6.6 source treeを読み取り専用で独立監査。
- **2026-07-25 / Reconstructed Public Version 1:** 保存された監査結果と照合報告からQ側の単独監査記録を再構成。再構成文書であること、Phase 0の範囲、未確定事項を明記。
