# Phase 0 独立ソース監査 — 挿入位置の特定 (VecTA)

**対象:** OpenClaw v2026.6.6 (tag commit `8c802aa6`)
**方式:** Qの監査結果を見る前に独立作成(封筒方式)。突き合わせ用。
**日付:** 2026-07-23

---

## A — Context Projection Guard の挿入位置

**第一候補(推奨): `transformContext` のラップ — 全provider callが通る単一の関門**

- 基底チェーン: `src/agents/sessions/sdk.ts` L450–456 — `transformContext` → extension runnerの `emitContext`(`src/agents/sessions/extensions/runner.ts` L912)。
- **前例:** `src/agents/embedded-agent-runner/tool-result-context-guard.ts` L342–355 が、まさにこのパターンで `mutableAgent.transformContext` をラップし、しかも `projectTranscriptPromptMessages` / `stripTranscriptPromptMarkers` により**「provider送信時だけの投影」を既に実装している**。A-forwardはこの前例の複製として書ける(canonical不変・送信直前で非terminal textを除去)。
- この一点で **same-run内の後続pass** と **turn跨ぎの履歴** の両方が一様に投影される(すべての呼び出しがここを通過するため)。Projection Decision Logの記録位置もここ。

**第二候補(初期パッチでは触らない・観測点として利用):**
- run開始時のreplay組み立て: `src/agents/embedded-agent-runner/run/attempt.ts` L2919 `sanitizeSessionHistory`(import L252–253)、近傍の `validateReplayTurns`。
- コンパクション後再構築: `src/agents/embedded-agent-runner/compact.ts` L1306 / L1321。
- transform境界での実装がこれらを包含するため、二重実装は不要。fixture検証の観測点として使う。

## B — Final-Only Delivery Gate の挿入位置

**配送発生点:**
- `src/agents/embedded-agent-subscribe.handlers.messages.ts` — text_end処理 L578–582 周辺(現行の抑制判定 `shouldSuppressAssistantVisibleOutput` は L45、**phaseベースのみ**)と、message_end flush L881–L1070 周辺。`blockReplyBreak` の既定は `"text_end"`(`embedded-agent-subscribe.ts` L175)。
- 追加すべき条件: assistant messageが `toolCalls` を持つ / `stopReason=toolUse` → 通常block replyとして出さない。

**蓄積側の防護(run終了payloadのfan-out防止):**
- `src/agents/embedded-agent-subscribe.ts` L444–456 `pushAssistantText` / L458–487 `finalizeAssistantTexts` — 非terminal textを `assistantTexts` に入れないガード。`assistantTexts` は `run.ts` L3018 以降で最終payloadへ変換される。

**T10(意図的多段送信)の裏付け:** `embedded-agent-subscribe.ts` L488以降に「Messaging tool duplicate detection」が既存 — **明示的send toolは別経路**として既に区別されており、Bが暗黙配送だけをゲートできる構造は6.6に元からある。

**流用できる既存テスト:** `…suppresses-commentary-phase-output.test.ts`、`…suppresses-message-end-block-replies-message-tool….test.ts`、`embedded-agent-subscribe.before-terminal-delivery.test.ts`。

## C — History Mirror Filter の挿入位置

- **ファイル:** `src/gateway/server-methods/chat.ts` — `ChatHistoryMethod`(`"chat.history" | "chat.startup"`)は L224。mirror判定の既存述語が L1625 / L1678 にあり、そのまま流用可能。
- **前例:** `src/gateway/session-utils.fs.ts` L1266–1270 — 統計/推定経路では `provider=="openclaw" && model=="delivery-mirror"` を**既に除外(return null)している**。Cは、この既存の除外パターンをchat.historyの応答組み立てへ複製するだけの低リスク変更。
- **未確定セル(正直に):** chat.ts内で通常messageリストを構築する正確な関数・行は未固定。ファイルと流用述語までが本監査の確定範囲。Qの結果との突き合わせ対象。

---

## 三点共通の所見

1. 三つとも**既存パターンの複製で書ける**(A=tool-result-context-guardの投影ラップ、B=既存suppress判定への条件追加、C=session-utilsの既存除外述語)。6.6のコードベース自体が、必要な形をすでに知っている。
2. AをtransformContextで実装すると、Decision Log・A-forward/A-retroのcutoff判定・fallback検出をすべて同じ関門に置ける — ハーネスの provider context projection 層の原型としてそのまま切り出せる。
3. 相違が出た場合の突き合わせ優先順位: Aの関門選択(transform境界 vs replay組み立て) > Bのゲート位置(handlers vs pushAssistantText) > Cの関数特定。
