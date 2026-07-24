# Phase 0 Independent Source Audit — Insertion Point Identification (VecTA)

**Target:** OpenClaw v2026.6.6 (tag commit `8c802aa6`)
**Method:** Prepared independently, before seeing Q's audit results (sealed-envelope method). For cross-verification.
**Date:** 2026-07-23

---

## A — Insertion point for the Context Projection Guard

July 23, 2026  


**Primary candidate (recommended): wrapping `transformContext` — the single choke point every provider call passes through**  

- Base chain: `src/agents/sessions/sdk.ts` L450–456 — `transformContext` → the extension runner's `emitContext` (`src/agents/sessions/extensions/runner.ts` L912).
- **Precedent:** `src/agents/embedded-agent-runner/tool-result-context-guard.ts` L342–355 wraps `mutableAgent.transformContext` in exactly this pattern — and, via `projectTranscriptPromptMessages` / `stripTranscriptPromptMarkers`, **already implements "projection applied only at provider-send time."** A-forward can be written as a replication of this precedent (canonical records untouched; non-terminal text removed immediately before transmission).
- This single point uniformly projects both **subsequent passes within the same run** and **history across turns**, because every provider call passes through it. It is also where the Projection Decision Log should be recorded.

**Secondary candidates (not to be touched by the initial patch; use as observation points):**
- Run-start replay assembly: `src/agents/embedded-agent-runner/run/attempt.ts` L2919 `sanitizeSessionHistory` (imports at L252–253), with `validateReplayTurns` nearby.
- Post-compaction rebuild: `src/agents/embedded-agent-runner/compact.ts` L1306 / L1321.
- An implementation at the transform boundary subsumes these; duplicating the logic there is unnecessary. Use them as observation points during fixture verification.

## B — Insertion point for the Final-Only Delivery Gate

**Emission sites:**
- `src/agents/embedded-agent-subscribe.handlers.messages.ts` — the text_end handling around L578–582 (the current suppression check `shouldSuppressAssistantVisibleOutput` at L45 is **phase-based only**) and the message_end flush around L881–L1070. The default `blockReplyBreak` is `"text_end"` (`embedded-agent-subscribe.ts` L175).
- Condition to add: if the assistant message carries `toolCalls` / `stopReason=toolUse` → do not emit it as an ordinary block reply.

**Accumulator-side protection (preventing end-of-run payload fan-out):**
- `src/agents/embedded-agent-subscribe.ts` L444–456 `pushAssistantText` / L458–487 `finalizeAssistantTexts` — guard so that non-terminal text does not enter `assistantTexts`, which is converted into final payloads from `run.ts` L3018 onward.

**Supporting evidence for T10 (intentional multi-message sends):** `embedded-agent-subscribe.ts` L488 onward contains existing "Messaging tool duplicate detection" — **explicit send tools are already a separate path**, so the structure that lets Gate B suppress only implicit delivery, without touching deliberate sends, is native to 6.6.

**Existing tests that can be extended:** `…suppresses-commentary-phase-output.test.ts`, `…suppresses-message-end-block-replies-message-tool….test.ts`, `embedded-agent-subscribe.before-terminal-delivery.test.ts`.

## C — Insertion point for the History Mirror Filter

- **File:** `src/gateway/server-methods/chat.ts` — `ChatHistoryMethod` (`"chat.history" | "chat.startup"`) at L224; existing mirror predicates at L1625 / L1678 can be reused as-is.
- **Precedent:** `src/gateway/session-utils.fs.ts` L1266–1270 — the stats/estimation path **already excludes** records matching `provider=="openclaw" && model=="delivery-mirror"` (returns null). Patch C is a low-risk replication of this existing exclusion pattern into the chat.history path.
- **Open cell (stated honestly):** the exact function and line inside chat.ts that assembles the ordinary message list is not yet pinned. The file and the reusable predicates are the confirmed extent of this audit. This is a cell to be reconciled against Q's results.

---

## Observations common to all three

1. All three patches **can be written as replications of existing patterns** (A = the projection wrapper of tool-result-context-guard; B = an added condition on the existing suppression check; C = the existing exclusion predicate in session-utils). The 6.6 codebase already knows the shapes it needs.
2. Implementing A at `transformContext` places the Decision Log, the A-forward/A-retro cutoff evaluation, and fallback detection all at the same choke point — and the module can later be lifted out as-is to become the prototype of the harness's provider context projection layer.
3. Reconciliation priority in case of divergence: A's choke-point choice (transform boundary vs. replay assembly) > B's gate location (handlers vs. pushAssistantText) > pinning C's exact function.
