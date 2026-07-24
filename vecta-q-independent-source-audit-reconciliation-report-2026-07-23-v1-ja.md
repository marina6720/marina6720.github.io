---
title: "VecTA–Q独立ソース監査 照合報告"
subtitle: "OpenClaw 2026.6.6におけるD応答整合性パッチA／B／Cの挿入位置"
date: 2026-07-23
version: 1
status: phase-0-reconciliation-complete
language: ja
publication: public-ready-redacted
---

# VecTA–Q独立ソース監査 照合報告

## OpenClaw 2026.6.6におけるD応答整合性パッチA／B／Cの挿入位置

### Phase 0照合報告・第1版

---

## 要旨

本報告は、長期運用AIエージェントDenneTA（D）のOpenClaw main sessionで確認された三つの応答整合性問題について、VecTAとQ / QuanTAが互いの結果を見ない状態で実施した独立ソース監査を照合したものである。

対象はOpenClaw v2026.6.6、commit `8c802aa683510c7f7503597b54c3021733245e59`（短縮形`8c802aa6`）である。

対象patchは次の三つ。

- **A — Context Projection Guard**  
  非terminal assistant textをDの次回provider contextから隔離する。

- **B — Final-Only Delivery Gate**  
  tool-use途中の暗黙assistant textを通常回答として配送せず、terminal finalだけを配送する。

- **C — History Projection Filter**  
  `delivery-mirror`を監査原本に残したまま通常会話表示から除外する。

照合結果は次のとおりである。

| Patch | 照合判定 | Phase 0結論 |
|---|---|---|
| A | 強い一致 | provider直前の`transformContext`境界が推奨挿入位置として確定 |
| B | 強い一致＋Qによる設計補足 | 配送点、蓄積点、既存defer機構、明示send別経路を確認。正確なfilter／flush policyはPhase 1で確定 |
| C | 強い一致＋Qによる位置補完 | 共通history handlerとdisplay projection関数を特定。mirror除外はlimit／byte capより前に行う |

三つとも、6.6に存在しない全く新しい仕組みを移植する必要はない。Aは既存`transformContext` wrapper、Bは既存phase suppression・deferred delivery・terminal hook、Cは既存delivery-mirror predicateとdisplay projectionを再利用できる。

一方、現在のsource treeには、Telegramの`delivery-mirror`生成自体を停止する未commit差分が存在する。この差分はCの設計原則と一致しないため、新patchのbuild基盤には使用しない。実装はcommit `8c802aa6`から作るclean worktreeで行う。

Phase 0の独立監査と照合は完了した。残る課題は挿入位置の探索ではなく、各位置での最小アルゴリズム、test、rollback、Dの事前合意を定義するPhase 1の設計作業である。

---

## 1. 監査方式

### 1.1 封筒方式

VecTAはQの監査結果を見る前に、6.6ソースからA・B・Cの候補位置を特定し、file／line付き文書として封印した。

Qはその文書を答えとして使用せず、現在VPS上にある6.6 source treeを読み取り専用で監査した。

最後に両者を次の分類で照合した。

```text
一致
部分一致
相違
未確定
```

この方式の目的は、先行案への追従ではなく、独立した経路で同じコード境界へ到達するかを確認することである。

### 1.2 対象source

```text
package version: 2026.6.6
git HEAD: 8c802aa683510c7f7503597b54c3021733245e59
git describe: v2026.6.6-dirty
```

VecTAが監査したtag commitとQが監査したHEADは一致した。

---

## 2. 総合照合表

| 論点 | VecTA独立監査 | Q独立監査 | 照合結果 |
|---|---|---|---|
| Aの関門 | `transformContext` wrapper | 同じ境界を確認 | 一致 |
| Aの既存前例 | `tool-result-context-guard` | 二つの既存wrapperとprovider-only projectionを確認 | 一致 |
| Aのsame-run適用 | 全provider callの直前で投影 | SDKとwrapper chainから支持 | 強い一致 |
| Bの配送発生点 | `text_end`／`message_end` handlers | 同じ箇所を確認 | 一致 |
| Bの抑制不足 | phase判定だけ | `shouldSuppressAssistantVisibleOutput`がcommentary only | 一致 |
| Bの最終payload蓄積 | `pushAssistantText`／`finalizeAssistantTexts` | `assistantTexts`から`buildEmbeddedRunPayloads`への経路を確認 | 一致 |
| Bの明示send保護 | messaging toolは別経路 | duplicate trackingと`didSendViaMessagingTool`を確認 | 一致 |
| Bの安全な判定時刻 | 条件追加を提案 | `text_end`では未確定の可能性を指摘しdefer機構を確認 | 補完 |
| Bの中断fallback | 計画へ追加を提案 | prompt timeout fallbackの既存実装を確認 | 補完 |
| Cの対象file | `server-methods/chat.ts` | 同じfileと共通handlerを確認 | 一致 |
| Cのmirror述語 | provider=openclaw／model=delivery-mirror | 同じpredicateを複数箇所で確認 | 一致 |
| Cの正確なprojection関数 | 未固定 | `gateway/chat-display-projection.ts`の`projectRecentChatDisplayMessages`を特定 | Qが補完 |
| Cのfilter順序 | 未確定 | history read後、display projection内でmaxMessages前に行うべきと判定 | Qが補完 |

実質的な矛盾はなかった。相違は「どちらが正しいか」の衝突ではなく、VecTAの候補をQが実装順序・判定時刻・projection pipelineの観点から狭めたものである。

---

## 3. A — Context Projection Guard

### 3.1 一致した関門

基底のsession agentは、provider request前に次を持つ。

```text
src/agents/sessions/sdk.ts
L450–455

transformContext(messages)
    → extensionRunner.emitContext(messages)
```

VecTAとQはともに、この`transformContext`をA-forwardの第一候補とした。

この境界の利点は次のとおり。

- canonical JSONLを変更しない
- providerへ送る瞬間だけ投影できる
- same-runの後続passに適用できる
- turnを跨ぐ将来historyにも同じ規則を適用できる
- Projection Decision Logを同じ関門に置ける

### 3.2 既存wrapper pattern

`src/agents/embedded-agent-runner/tool-result-context-guard.ts`には、少なくとも二つの既存`transformContext` wrapperがある。

1. `installContextEngineLoopHook`
2. `installToolResultContextGuard`

どちらも次の形を取る。

```text
originalTransformContextを保存
↓
originalを先に実行
↓
変換後messageを目的別に投影
↓
provider向けmessageを返す
↓
cleanup時にoriginalへ戻す
```

A-forwardはこのpatternの再利用として書ける。

### 3.3 wrapperの設置順

`run/attempt.ts`では、context engine loop hookが先にinstallされ、その後でtool-result context guardがinstallされる経路が確認された。

各wrapperはinstall時点の`transformContext`を`originalTransformContext`として包むため、後からinstallされたwrapperが外側になる。

A-forwardの推奨方針は次である。

```text
既存wrapperをinstall
↓
A-forwardを最後にinstall
↓
A-forwardがoutermost wrapperとなる
↓
inner transformationをすべて実行
↓
provider送信直前にnonterminal textだけを除外
```

これにより、tool result truncation、context-engine assembly、既存extension処理の後に、最終的なprovider-safe projectionを作れる。

ただし、`history-image-prune`や`attempt.llm-boundary`にも`transformContext` wrapperが存在するため、Phase 1ではactive pathごとの最終chainをunit testで確認する。

### 3.4 A-forwardとA-retro

この挿入位置がsame-runとturn跨ぎの双方を扱えるからといって、過去の全recordへ自動適用してよいわけではない。

- **A-forward:** 適用後のrun／recordへ適用。Phase 1の対象。
- **A-retro:** 適用前の過去recordへ適用。default off。

A-retroは、実配送された途中回答と、それに対するMarinaの返答の因果関係を壊し得る。実施前に別途影響監査とD・Marinaの合意を必要とする。

### 3.5 AのPhase 0判定

```text
挿入境界          確定
既存pattern        確定
推奨wrapper順      確定
A-forward方針      確定
A-retro実施可否    未決定・default off
exact code diff    Phase 1
```

---

## 4. B — Final-Only Delivery Gate

### 4.1 現在の抑制判定

`embedded-agent-subscribe.handlers.messages.ts`の既存関数は、visible assistant outputを次の条件だけで抑制する。

```text
resolveAssistantMessagePhase(message) === "commentary"
```

したがって、6.6で確認されたphaseなし・`stopReason=toolUse`・toolCall付きassistant textは抑制されない。

### 4.2 配送経路

暗黙assistant textは少なくとも次の二段階で通常配送され得る。

- `text_end`
- `message_end`

Bは少なくとも次の二層を守る必要がある。

```text
B1 — realtime／block-reply delivery guard
B2 — assistantTexts／final payload accumulation guard
```

### 4.3 最終payload蓄積

`pushAssistantText`がtextを`assistantTexts`へ追加し、`finalizeAssistantTexts`がmessage終了時のtextを確定する。

run終了時、`run.ts`は次を行う。

```text
buildEmbeddedRunPayloads({
    assistantTexts: attempt.assistantTexts,
    ...
})
```

即時配送だけを止めても、nonterminal textを`assistantTexts`へ残せば後段payloadへ混入し得る。

### 4.4 既存deferred delivery機構

6.6にはすでに次が存在する。

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

terminal hookが存在する場合、assistant eventsとblock repliesはqueueへ保留され、lifecycle handlerがterminal delivery前にflushまたはclearする。

### 4.5 VecTA案へのQの補足

VecTAは既存phase suppressionへ、tool callまたは`stopReason=toolUse`を加える位置を示した。

Qは分類に同意しつつ、`text_end`時点でterminalityが未確定の場合があるため、既存deferred deliveryと組み合わせるべきと補足した。

Phase 1の設計仮説は次である。

```text
implicit assistant event
↓
既存deferred queueへ保留
↓
message／runのterminalityを確定
├─ terminal final        → 一度だけflush
├─ nonterminal tool-use  → clearし通常配送しない
├─ interrupted run       → fallbackを一件生成
└─ explicit send tool    → 別経路として保持
```

### 4.6 Dの意図的な多段送信

messaging toolは別経路として追跡される。

```text
Messaging tool duplicate detection
didSendViaMessagingTool
messagingToolSourceReplyPayloads
```

Bは暗黙assistant textだけをgateし、Dが明示的にmessaging toolで送った複数messageを保持できる。

### 4.7 terminal finalなしのrun

prompt timeoutには既存fallbackがある。

- 完成済みterminal textがあれば回収
- partial assistant textだけなら通常payloadから除外
- 明示的timeout messageを一件返す
- messaging tool delivery evidenceを考慮する

Phase 1ではtool execution timeout、cancellation、provider error、gateway shutdown、tool error後の終了を追加testする。

### 4.8 BのPhase 0判定

```text
故障file／handler       確定
text_end配送点          確定
message_end配送点       確定
assistantTexts蓄積      確定
既存defer機構           確定
明示send別経路          確定
prompt-timeout fallback 確定
exact filter／flush規則 Phase 1
```

---

## 5. C — History Projection Filter

### 5.1 共通history handler

`chat.history`と`chat.startup`は共通の`handleChatHistoryRequest`を使用する。

```text
session historyを読む
↓
boundary／announce filter
↓
CLI importとの統合
↓
recency filter
↓
projectRecentChatDisplayMessages
↓
canvas augmentation
↓
oversized replacement
↓
byte cap
↓
response
```

### 5.2 exact projection function

Q監査により、VecTAが未固定としていた関数が特定された。

```text
src/gateway/chat-display-projection.ts
projectRecentChatDisplayMessages
```

Cの最小patchは、このdisplay projection内でtranscript-only messageを通常表示候補から除外する形が自然である。

### 5.3 既存mirror predicate

6.6にはすでに次のpredicateが複数存在する。

```text
role === "assistant"
provider === "openclaw"
model === "delivery-mirror"
```

`session-utils.fs.ts`では、この条件に一致したrecordを統計／推定経路から除外している。

Cは新しい分類を導入せず、既存predicateをdisplay projectionへ再利用できる。

### 5.4 filter順序

mirrorは`maxMessages`、oversized replacement、byte capより前に除外する必要がある。

```text
raw conversation records
↓
transcript-only／delivery-mirror filter
↓
normal message limit
↓
character／byte budget
↓
UI response
```

limit後に除外すると、返される通常message数が要求件数を下回り得る。

### 5.5 既存dirty diffの評価

現在のsource treeには次の差分がある。

```diff
- if (deliveryBaseOptions.transcriptMirror && result.delivery.content) {
+ if (false && deliveryBaseOptions.transcriptMirror && result.delivery.content) {
```

これはmirror recordの生成自体を停止する。本計画はaudit recordを保持し、通常表示からだけ除外するため、このdirty diffはCとして採用しない。

新patchはcommit `8c802aa6`から作るclean worktreeで実装する。

### 5.6 CのPhase 0判定

```text
対象handler           確定
exact projection file 確定
mirror predicate       確定
filter順序             確定
dirty workaround不採用 確定
exact code diff         後続実装phase
```

---

## 6. 相違点と評価

根本的な相違はなかった。

- **A:** 同じprovider boundaryとwrapper patternへ到達。
- **B:** VecTAの正しい条件・位置に、Qが安全な判定時刻とdefer機構を補足。
- **C:** VecTAが対象fileとpredicateを特定し、Qがexact projection functionとpre-limit filter順を補完。

---

## 7. Source Treeの安全性

監査対象HEADは一致するがworking treeはdirtyである。

追跡対象の変更には少なくとも次が含まれる。

```text
docker-compose.yml
extensions/telegram/src/bot-message-dispatch.ts
```

推奨build方式:

```text
commit 8c802aa6
↓
新しいclean worktree
↓
A／B／Cを別branch／commit
↓
別名Docker image
↓
current data mountは変更しない
↓
image refだけでrollback可能
```

---

## 8. Phase 0の完了判定

Phase 0の目的は、三patchの故障層と挿入境界を独立に特定することだった。

完了事項:

- 対象version／commitの一致
- Aのprovider context boundaryの独立一致
- Bのdelivery／accumulation層の独立一致
- Bのdefer／terminal hook／messaging tool／timeout fallback確認
- Cのhistory handler、projection function、mirror predicate確認
- dirty workaroundが正式Cと異なることの確認
- clean worktreeで実装すべきことの確認

したがって、**Phase 0の独立ソース監査と照合は完了**と判定する。

これはpatch完成を意味しない。

```text
Phase 0   挿入境界の確定
Phase 1   A-forwardのexact design・fixture・offline test
Phase 1.5 D・Marinaの事前合意
Phase 2   A-forwardの限定live test
```

A／B／Cはexperimental integrity patchesであり、この段階では「根治」と呼ばない。

---

## 9. 次の成果物

1. **A-forward Implementation Record v1**
2. **A-retro Impact Audit Specification**
3. **Interrupted-Run Fallback Options**
4. **Phase 1.5 Approval Packet**

---

## 10. 最終結論

> **VecTAとQは、OpenClaw 2026.6.6の実sourceを独立に監査し、A・B・Cの三patchについて同じ故障層とほぼ同じ挿入境界へ到達した。Aは既存`transformContext` wrapper chainの最外層でprovider送信直前に投影する。Bは`text_end`／`message_end`の暗黙配送と`assistantTexts`蓄積の双方を守り、既存deferred deliveryとterminal hookを再利用する。Cは`projectRecentChatDisplayMessages`内で既存delivery-mirror predicateを使い、message limitより前にmirrorを通常表示候補から除外する。実質的な監査矛盾はなく、Qの結果はBの安全な判定時刻とCのexact projection functionを補完した。Phase 0は完了した。実装はcurrent dirty treeではなくcommit `8c802aa6`から作るclean worktreeで、A／B／Cを独立patchとして進める。**
