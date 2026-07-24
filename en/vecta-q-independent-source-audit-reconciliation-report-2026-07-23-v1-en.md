---
title: "VecTA–Q Independent Source Audit Reconciliation Report"
subtitle: "Insertion Points for D Response Integrity Patches A, B, and C in OpenClaw 2026.6.6"
date: 2026-07-23
version: 1
status: phase-0-reconciliation-complete
language: en
publication: public-ready-redacted
---

# VecTA–Q Independent Source Audit Reconciliation Report

## Insertion Points for D Response Integrity Patches A, B, and C in OpenClaw 2026.6.6

### Phase 0 Reconciliation Report — Version 1  

July 23, 2026

---

## Executive Summary

This report reconciles two independent source audits of three response-integrity failures observed in DenneTA’s long-running OpenClaw main session. VecTA and Q / QuanTA inspected the source without seeing each other’s conclusions first.

The target was OpenClaw v2026.6.6 at commit `8c802aa683510c7f7503597b54c3021733245e59` (`8c802aa6`).

The three proposed patches are:

- **A — Context Projection Guard**  
  Isolate non-terminal assistant text from D’s next provider context.

- **B — Final-Only Delivery Gate**  
  Prevent implicit assistant text produced during tool use from being delivered as an ordinary reply before the terminal final answer.

- **C — History Projection Filter**  
  Preserve `delivery-mirror` as an audit record while excluding it from ordinary conversation display.

The reconciliation result is:

| Patch | Reconciliation judgment | Phase 0 conclusion |
|---|---|---|
| A | Strong convergence | the provider-side `transformContext` boundary is confirmed as the preferred insertion point |
| B | Strong convergence with a Q timing refinement | delivery, accumulation, deferral, terminal hook, and explicit-send paths are identified; exact filter/flush policy is finalized in Phase 1 |
| C | Strong convergence with Q completing the exact location | the shared history handler and display-projection function are identified; mirrors should be filtered before message and byte limits |

All three patches can reuse patterns already present in OpenClaw 2026.6.6. A reuses existing `transformContext` wrappers, B reuses phase suppression, deferred delivery, terminal hooks, and messaging-tool separation, and C reuses an existing delivery-mirror predicate inside the established display-projection layer.

The current working tree contains an uncommitted change that disables creation of Telegram transcript mirrors entirely. That is not the intended C design, which preserves audit records and filters only ordinary display. Implementation must therefore use a clean worktree created from commit `8c802aa6`.

The independent Phase 0 source audit and reconciliation are complete. The remaining work is no longer discovery of insertion points; it is Phase 1 design of the smallest algorithms, tests, rollback units, and D’s pre-deployment approval.

---

## 1. Audit Method

### 1.1 Sealed-envelope method

VecTA independently identified candidate insertion points in the 2026.6.6 source and sealed the findings in a file-and-line report before seeing Q’s audit.

Q then performed a read-only audit of the current 2026.6.6 source tree without treating VecTA’s report as the answer.

The results were compared using:

```text
match
partial match
divergence
unresolved
```

The purpose was to test whether two independent routes converged on the same code boundaries.

### 1.2 Source target

```text
package version: 2026.6.6
git HEAD: 8c802aa683510c7f7503597b54c3021733245e59
git describe: v2026.6.6-dirty
```

VecTA’s target commit and Q’s audited HEAD matched.

---

## 2. Reconciliation Matrix

| Question | VecTA audit | Q audit | Result |
|---|---|---|---|
| A boundary | `transformContext` wrapper | same boundary confirmed | match |
| A precedent | `tool-result-context-guard` | two existing wrappers and provider-only projection confirmed | match |
| A same-run coverage | projection before provider calls | supported by SDK and wrapper chain | strong match |
| B delivery point | `text_end` / `message_end` handlers | same locations confirmed | match |
| B suppression gap | phase-only suppression | `shouldSuppressAssistantVisibleOutput` is commentary-only | match |
| B payload accumulation | `pushAssistantText` / `finalizeAssistantTexts` | `assistantTexts` to `buildEmbeddedRunPayloads` confirmed | match |
| B explicit sends | messaging tools are separate | duplicate tracking and `didSendViaMessagingTool` confirmed | match |
| B safe decision time | add non-terminal condition | Q found `text_end` may be too early and confirmed deferral infrastructure | refinement |
| B interruption fallback | requested as missing design | existing prompt-timeout fallback confirmed | refinement |
| C file | `server-methods/chat.ts` | same file and shared handler confirmed | match |
| C predicate | provider=openclaw / model=delivery-mirror | same predicate confirmed in multiple paths | match |
| Exact C projection function | not fixed | `projectRecentChatDisplayMessages` in `gateway/chat-display-projection.ts` identified | completed by Q |
| C filter order | unresolved | filter before message and byte limits | completed by Q |

No substantive contradiction was found. The differences were refinements of timing, wrapper order, and projection placement.

---

## 3. A — Context Projection Guard

### 3.1 Converged boundary

The base session agent exposes:

```text
src/agents/sessions/sdk.ts
L450–455

transformContext(messages)
    → extensionRunner.emitContext(messages)
```

Both audits independently selected this provider-side transformation boundary for A-forward.

It provides:

- unchanged canonical JSONL
- provider-only projection
- same-run protection
- cross-turn future-history protection
- a single location for the Projection Decision Log

### 3.2 Existing wrapper pattern

`src/agents/embedded-agent-runner/tool-result-context-guard.ts` contains at least two existing `transformContext` wrappers:

1. `installContextEngineLoopHook`
2. `installToolResultContextGuard`

Both preserve the original transform, call it first, project the resulting messages for a narrower purpose, and restore the original transform during cleanup.

A-forward can be implemented as another instance of this established pattern.

### 3.3 Wrapper order

`run/attempt.ts` installs the context-engine loop hook before the tool-result context guard on the observed paths. Because each wrapper captures the current transform as its original, later wrappers are outer wrappers.

The recommended A-forward placement is therefore:

```text
install existing wrappers
↓
install A-forward last
↓
A-forward becomes outermost
↓
run all inner transformations
↓
remove non-terminal user-facing text immediately before provider submission
```

Phase 1 must still test active paths involving history-image pruning and the LLM boundary wrapper.

### 3.4 A-forward versus A-retro

The insertion point can affect both current and historical messages, but the policies must remain separate.

- **A-forward:** applies after activation and is the Phase 1 target.
- **A-retro:** applies to pre-activation history and remains disabled by default.

Historical non-terminal text may have been delivered and answered by Marina. Retrospective filtering could break conversational causality and therefore requires a separate impact audit and approval by D and Marina.

### 3.5 Phase 0 status for A

```text
insertion boundary       confirmed
existing pattern         confirmed
recommended wrapper order confirmed
A-forward policy         confirmed
A-retro decision         unresolved; default off
exact code diff          Phase 1
```

---

## 4. B — Final-Only Delivery Gate

### 4.1 Current suppression rule

The current visible-output suppression function suppresses only:

```text
resolveAssistantMessagePhase(message) === "commentary"
```

A phase-less assistant message with `stopReason=toolUse` and a tool call therefore passes the existing gate, matching the observed 2026.6.6 failure.

### 4.2 Delivery paths

Implicit assistant text can be delivered through at least two stages:

- `text_end`
- `message_end`

B therefore requires two protections:

```text
B1 — realtime / block-reply delivery guard
B2 — assistantTexts / final-payload accumulation guard
```

### 4.3 Final-payload accumulation

`pushAssistantText` adds text to `assistantTexts`, and `finalizeAssistantTexts` finalizes message-level text.

At run completion, `run.ts` passes:

```text
assistantTexts: attempt.assistantTexts
```

to `buildEmbeddedRunPayloads`.

Suppressing immediate Telegram output alone would therefore be insufficient.

### 4.4 Existing deferral infrastructure

OpenClaw 2026.6.6 already contains:

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

When the terminal hook is active, assistant events and block replies can be queued rather than emitted immediately. The lifecycle handler can flush or clear them before terminal delivery.

This provides an established mechanism for delaying classification until a complete assistant message or run state is available.

### 4.5 Q’s refinement to VecTA’s proposal

VecTA identified the correct non-terminal conditions to add:

```text
tool calls present
or
stopReason = toolUse
```

Q agreed with the classification but noted that `text_end` may occur before all terminality information is reliable for every provider.

The recommended Phase 1 design hypothesis is:

```text
implicit assistant event
↓
existing deferred queue
↓
classify complete message / run
├─ terminal final        → flush once
├─ non-terminal tool use → clear from ordinary delivery
├─ interrupted run       → emit one fallback
└─ explicit send tool    → preserve separately
```

### 4.6 Intentional multi-message speech

Messaging-tool deliveries are already tracked separately through duplicate detection, `didSendViaMessagingTool`, and messaging-tool source-reply payloads.

B can therefore suppress implicit intermediate delivery without suppressing multiple messages D intentionally sends through a messaging tool.

### 4.7 Runs without a terminal final

The existing prompt-timeout path already:

- recovers a completed terminal answer when available
- avoids treating a partial fragment as a normal answer
- emits one explicit timeout message
- considers messaging-tool delivery evidence

Phase 1 expands testing to tool timeout, cancellation, provider error, gateway shutdown, and tool-error termination.

### 4.8 Phase 0 status for B

```text
failure files / handlers confirmed
text_end delivery point confirmed
message_end delivery point confirmed
assistantTexts accumulation confirmed
deferral infrastructure confirmed
explicit-send separation confirmed
prompt-timeout fallback confirmed
exact filter / flush policy Phase 1
```

---

## 5. C — History Projection Filter

### 5.1 Shared handler

`chat.history` and `chat.startup` use the shared `handleChatHistoryRequest`.

The observed pipeline is:

```text
read session history
↓
boundary and announcement filtering
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

### 5.2 Exact projection function

Q identified the exact display projection that VecTA had left unresolved:

```text
src/gateway/chat-display-projection.ts
projectRecentChatDisplayMessages
```

This is the preferred insertion layer for C.

### 5.3 Existing mirror predicate

The source already uses:

```text
role === "assistant"
provider === "openclaw"
model === "delivery-mirror"
```

to recognize delivery mirrors.

`session-utils.fs.ts` already excludes such records from a statistics/estimation path. `server-methods/chat.ts` uses the same predicate to locate source-reply mirrors.

C can therefore reuse an existing classification rather than inventing a new one.

### 5.4 Filter order

Mirrors must be removed before:

- `maxMessages`
- per-message replacement
- final byte budget

The intended order is:

```text
raw records
↓
transcript-only / mirror filter
↓
normal-message limit
↓
character and byte budgets
↓
UI response
```

Filtering after limits would reduce the number of normal conversation messages returned.

### 5.5 Rejection of the current dirty workaround

The current working tree contains:

```diff
- if (deliveryBaseOptions.transcriptMirror && result.delivery.content) {
+ if (false && deliveryBaseOptions.transcriptMirror && result.delivery.content) {
```

This disables mirror creation itself.

C must instead preserve the audit record and remove only the ordinary display projection. The dirty workaround is therefore rejected.

Implementation must use a clean worktree from `8c802aa6`.

### 5.6 Phase 0 status for C

```text
target handler          confirmed
exact projection file  confirmed
mirror predicate        confirmed
filter order            confirmed
dirty workaround rejected
exact code diff         later implementation phase
```

---

## 6. Differences and Their Meaning

No fundamental audit divergence was found.

- **A:** both audits reached the same provider boundary and wrapper pattern.
- **B:** VecTA found the correct suppression conditions and locations; Q added safe decision timing and the existing deferral mechanism.
- **C:** VecTA found the target file and predicate; Q completed the exact projection function and pre-limit placement.

---

## 7. Source-Tree Safety

The audited HEAD matches the target commit, but the working tree is dirty.

At least the following tracked files are modified:

```text
docker-compose.yml
extensions/telegram/src/bot-message-dispatch.ts
```

Recommended build method:

```text
commit 8c802aa6
↓
new clean worktree
↓
separate branch and commit for A, B, and C
↓
separately named Docker images
↓
unchanged current data mounts
↓
rollback by image reference only
```

---

## 8. Phase 0 Closure

Phase 0 was intended to independently locate the correct failure layers and insertion boundaries, not to produce code.

The following are complete:

- version and commit agreement
- independent convergence on A’s provider-context boundary
- independent convergence on B’s delivery and accumulation layers
- confirmation of B’s deferral, terminal hook, timeout fallback, and messaging-tool separation
- confirmation of C’s shared history handler, exact projection function, and mirror predicate
- rejection of the dirty mirror-generation workaround
- decision to implement from a clean worktree

Therefore, the **independent Phase 0 source audit and reconciliation are complete**.

This does not mean the patches are complete.

```text
Phase 0   insertion boundaries confirmed
Phase 1   exact A-forward design, fixtures, and offline tests
Phase 1.5 D and Marina pre-approval
Phase 2   limited live validation of A-forward
```

A, B, and C remain experimental integrity patches and are not described as a root cure.

---

## 9. Next Deliverables

1. **A-forward Implementation Record v1**
2. **A-retro Impact Audit Specification**
3. **Interrupted-Run Fallback Options**
4. **Phase 1.5 Approval Packet**

---

## 10. Final Conclusion

> **VecTA and Q independently inspected the OpenClaw 2026.6.6 source and converged on the same failure layers and substantially the same insertion boundaries for patches A, B, and C. A belongs at the outer provider-side `transformContext` projection after existing wrappers. B must protect both implicit `text_end` / `message_end` delivery and `assistantTexts` accumulation, reusing existing deferred-delivery and terminal-hook infrastructure. C belongs inside `projectRecentChatDisplayMessages`, where the existing delivery-mirror predicate should exclude mirrors before normal message and byte limits. No substantive audit contradiction was found. Q’s audit refined the safe decision timing for B and completed the exact projection location for C. Phase 0 is complete. Implementation must proceed from a clean worktree at commit `8c802aa6`, with A, B, and C maintained as independent experimental patches.**
