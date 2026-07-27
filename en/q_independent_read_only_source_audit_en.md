|   |
|---|
|title: “Q Independent Read-Only Source Audit — Phase 0”|
|subtitle: “Insertion Boundaries for D Response Integrity Patches A, B, and C in OpenClaw 2026.6.6”|
|audit_date: 2026-07-23|
|record_created: 2026-07-25|
|version: 1|
|status: reconstructed-phase-0-audit-record|
|language: en|
|publication: public-ready-redacted|

# Q Independent Read-Only Source Audit — Phase 0

## Insertion Boundaries for D Response Integrity Patches A, B, and C in OpenClaw 2026.6.6

### Reconstructed Public Record — Version 1

---

## Positioning of This Document

On July 23, 2026, Q / QuanTA conducted a read-only examination of the OpenClaw 2026.6.6 source tree without treating the findings of VecTA’s independent audit as an answer.

However, Q’s standalone audit findings were not frozen and preserved as an independent document on that day. This document is a public record reconstructed on July 25, 2026 from preserved source-audit findings, file and function identifiers, the reconciliation report, the subsequent Implementation Record, and design judgments established during the audit.

This is therefore not a verbatim restoration of the original primary record. Nor does it create new conclusions retrospectively to match the later implementation. It reorganizes the observations, inferences, and unresolved questions that Q had independently reached during Phase 0 while preserving their evidential relationships.

---

## Abstract

In the long-running OpenClaw main session of the AI agent DenneTA (D), an apparent conflation was observed among three layers involving assistant text generated during tool use:

1.      Context reintroduced into the next provider call

2.      Ordinary user-facing delivery through Telegram or another channel

3.      Ordinary conversation-history display

To separate these layers without modifying the canonical transcript, Phase 0 investigated the insertion boundaries for three experimental integrity patches.

- **A — Context Projection Guard**  
    Isolate non-terminal assistant text only from D’s next provider context.
- **B — Final-Only Delivery Gate**  
    Prevent implicit assistant text generated during tool use from being delivered as an ordinary reply, and deliver the terminal final answer exactly once.
- **C — History Projection Filter**Preserve delivery-mirror in the canonical audit record while excluding it from ordinary conversation display.

Q’s audit reached the following conclusions:

|Patch|Q’s Phase 0 conclusion|
|---|---|
|A|The transformContext wrapper boundary immediately before provider submission is the smallest viable insertion candidate|
|B|Both text_end / message_end delivery and assistantTexts accumulation must be protected|
|C|Mirrors should be excluded inside projectRecentChatDisplayMessages, before the normal message limit is applied|
|Source tree|The current dirty tree should not be used as the build base; a clean worktree should be created from commit 8c802aa6|

Phase 0 was an audit of insertion boundaries. It did not complete the patches or authorize live operation.

---

## 1. Audit Conditions

### 1.1 Independence

VecTA first identified candidate locations in the OpenClaw 2026.6.6 source and sealed its audit findings with file and line references.

Q did not use those findings as an answer. Instead, Q independently examined the source tree on the VPS through a separate read-only path. The two sets of findings were opened and compared only after Q’s audit had been completed.

### 1.2 Read-Only Scope

The following actions were not performed during Phase 0:

·         Editing source files

·         Changing OpenClaw configuration

·         Restarting the Gateway

·         Sending input to D’s main session

·         Modifying the canonical JSONL

·         Writing to the workspace or memory files

·         Resetting, stashing, or cleaning the dirty tree

·         Building or switching Docker images

The objective was not to apply a repair. It was to identify the failure layers, existing safety mechanisms, and smallest viable insertion boundaries.

### 1.3 Target Source

package version: 2026.6.6  
git HEAD: 8c802aa683510c7f7503597b54c3021733245e59  
git describe: v2026.6.6-dirty

### 1.4 Categories of Judgment

The audit distinguished three types of information:

·         **Observed fact:** A function, call path, predicate, or installation order directly confirmed in the source

·         **Design inference:** A candidate for the smallest patch derived from the existing structure

·         **Unresolved:** A matter that should not be decided without Phase 1 tests, D’s judgment, or live validation

When a record was ambiguous, the default was not to speculate toward removal, but to preserve it safely.

---

## 2. Central Audit Principle

The three patches must not be treated as a single display problem.

canonical transcript        │  
        ├── provider context projection  ── A  
        ├── user-facing delivery         ── B  
        └── ordinary history display     ── C

The protected interests extend beyond eliminating visible duplication.

·         Do not lose the canonical record

·         Do not break the correspondence between a toolCall and its toolResult

·         Preserve the terminal final answer

·         Do not confuse a message explicitly sent by D with implicit assistant output

·         Do not retrospectively alter the causal structure of earlier conversation

·         Do not over-exclude records whose status is uncertain

·         Make rollback possible without data migration

---

## 3. A — Context Projection Guard

### 3.1 Audit Question

When a non-terminal assistant message contains both text and a toolCall, where can only the provider-facing message be safely projected without changing the canonical JSONL?

canonical:  
assistant [text, toolCall]  
↓  
toolResult  
↓  
next provider call

The target behavior is:

canonical record:    [text, toolCall]  
provider projection: [toolCall]

### 3.2 transformContext Immediately Before the Provider Boundary

The base session agent has a context-transformation boundary before a provider request is submitted.

src/agents/sessions/sdk.ts  
  
transformContext(messages)  
    → extensionRunner.emitContext(messages)

This boundary satisfies the following conditions:

·         It does not modify the canonical JSONL

·         It can project only the provider-facing messages

·         It can affect later provider calls within the same run

·         It can apply the same rule to future history across turns

·         It can generate a Decision Log at the same boundary as the provider projection

Q therefore selected this as the primary candidate boundary for A-forward.

### 3.3 Existing Wrapper Pattern

An existing transformContext wrapper pattern was present in:

src/agents/embedded-agent-runner/tool-result-context-guard.ts

The relevant functions included:

·         installContextEngineLoopHook

·         installToolResultContextGuard

The basic pattern was:

save the current transformContext  
↓  
run the saved transform first  
↓  
project the returned messages for a limited purpose  
↓  
return the provider-facing messages  
↓  
restore the original transform during cleanup

Q concluded that A-forward could be implemented by reusing this established pattern rather than introducing an entirely new session mechanism into OpenClaw.

### 3.4 Wrapper Order

Along the observed path in run/attempt.ts, the tool-result context guard was installed after the context-engine loop hook.

Because each wrapper encloses the transform that exists at installation time, the wrapper installed later becomes the outer wrapper.

The Phase 0 recommended order was:

install the existing wrappers  
↓  
install A-forward last  
↓  
make A-forward the outermost wrapper  
↓  
run every inner transformation  
↓  
apply the limited projection immediately before provider submission

Other wrappers could be added on different active paths, so the final order was left for confirmation through a Phase 1 wrapper-chain test.

### 3.5 Separation of A-forward and A-retro

The fact that the same insertion boundary could also affect historical messages does not mean that pre-activation history should automatically be changed.

·         **A-forward:** Applies to records created after activation

·         **A-retro:** Applies to historical records created before activation

A-retro should remain off by default because earlier non-terminal text may actually have been delivered to Marina and may have received a reply from her. Removing only that text from provider history afterward could alter the causal structure of the conversation.

### 3.6 Phase 0 Judgment for A

provider projection boundary  confirmed  
existing wrapper pattern      confirmed  
outermost-placement hypothesis confirmed  
A-forward / A-retro separation confirmed  
exact classifier               Phase 1  
exact code diff                Phase 1  
fixtures / tests               Phase 1  
live activation                not performed

---

## 4. B — Final-Only Delivery Gate

### 4.1 Audit Question

Why could non-terminal text generated during tool use be delivered as an ordinary reply and also enter the final payload? Is stopping delivery alone sufficient?

### 4.2 Existing Suppression Condition

The existing suppression logic in embedded-agent-subscribe.handlers.messages.ts primarily checked the following condition for visible assistant output:

resolveAssistantMessagePhase(message) === "commentary"

Assistant text observed under OpenClaw 2026.6.6 that had no explicit phase, used stopReason=toolUse, and contained a tool call would not be suppressed by this condition alone.

### 4.3 Two Failure Layers

Implicit assistant text has at least two paths:

B1 — ordinary delivery at text_end / message_end  
B2 — accumulation in assistantTexts and run-final payload construction

The source showed that pushAssistantText and finalizeAssistantTexts aggregate text into assistantTexts, which is later passed to buildEmbeddedRunPayloads.

Stopping real-time delivery alone is therefore insufficient. If the text remains in assistantTexts, it may still enter the final payload. B must protect both delivery and accumulation.

### 4.4 Existing Deferred-Delivery Mechanism

OpenClaw 2026.6.6 already contained structures that could be reused to defer output until terminality was known.

deferBlockReplyDelivery  
deferredAssistantEvents  
deferredBlockReplies  
flushDeferredAssistantEvents  
flushDeferredBlockReplies  
clearDeferredAssistantEvents  
clearDeferredBlockReplies  
onBeforeTerminalDelivery

These mechanisms make it possible to avoid classifying a message immediately upon receipt and instead flush or clear it later in the lifecycle.

### 4.5 Why text_end May Be Too Early

The presence of a tool call or stopReason=toolUse is important evidence of non-terminality. Depending on the provider and stream shape, however, the terminal status of the complete message may not yet be known at text_end.

Q therefore concluded that it would be safer not merely to add a condition directly at text_end, but to place the output in the existing deferred queue and classify it after the complete message or run state had become sufficiently clear.

implicit assistant event  
↓  
deferred queue  
↓  
establish message / run state  
├─ terminal final        → flush exactly once  
├─ non-terminal tool use → clear  
├─ interrupted run       → fallback  
└─ explicit send tool    → preserve as a separate path

### 4.6 Separation of Explicit Messaging Tools

When D intentionally sends multiple messages through a messaging tool, this is not equivalent to implicit assistant text.

The source contained separate paths and evidence including:

messaging-tool duplicate detection  
didSendViaMessagingTool  
messagingToolSourceReplyPayloads

B must gate implicit output while preserving D’s explicit sending actions.

### 4.7 Runs Without a Terminal Final Answer

The prompt-timeout path already contained fallback behavior:

·         Recover completed terminal text when available

·         Do not treat a partial fragment as an ordinary answer

·         Return one timeout message

·         Consider evidence that a message was sent through a messaging tool

From this observation, Q concluded that B should not be simplified to “return nothing when no final answer exists.” Timeout, cancellation, provider error, Gateway shutdown, and tool error must be treated as separate fixtures.

### 4.8 Phase 0 Judgment for B

text_end delivery point       confirmed  
message_end delivery point    confirmed  
assistantTexts accumulation   confirmed  
existing deferral mechanism   confirmed  
terminal hook                 confirmed  
messaging-tool separate path  confirmed  
prompt-timeout fallback       confirmed  
exact filter / flush rules    Phase 1  
interruption fixtures         Phase 1

---

## 5. C — History Projection Filter

### 5.1 Audit Question

Where is the smallest boundary at which delivery-mirror can remain in the canonical transcript as an audit record while no longer appearing as a duplicate message in ordinary conversation history?

### 5.2 Shared History Pipeline

Both chat.history and chat.startup used the shared handleChatHistoryRequest.

The observed pipeline was:

read session history  
↓  
boundary / announcement filtering  
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

### 5.3 Exact Display-Projection Function

Q identified the exact function that projects messages into ordinary display:

src/gateway/chat-display-projection.ts  
projectRecentChatDisplayMessages

The natural design for C is not to stop creation of mirror records, but to exclude them only from the ordinary display candidates inside this projection.

### 5.4 Existing Mirror Predicate

OpenClaw 2026.6.6 already used the following conditions in multiple places to identify a delivery mirror:

role === "assistant"  
provider === "openclaw"  
model === "delivery-mirror"

In session-utils.fs.ts, records matching this condition were excluded from statistical and estimation paths.

C can therefore reuse the existing predicate inside the display projection rather than creating a new and ambiguous classification.

### 5.5 Filter Order

Mirrors must be excluded before the following operations:

·         maxMessages

·         Oversized-message replacement

·         Final byte cap

The recommended order is:

raw records  
↓  
transcript-only / delivery-mirror filter  
↓  
normal-message limit  
↓  
character / byte budget  
↓  
UI response

If mirrors are removed only after the message limit is applied, they may first occupy part of the requested limit, reducing the number of ordinary messages returned to the user.

### 5.6 Rejection of the Dirty Workaround

At the time of the audit, the working tree contained an uncommitted difference that effectively disabled creation of mirror records:

- if (deliveryBaseOptions.transcriptMirror && result.delivery.content) {  
+ if (false && deliveryBaseOptions.transcriptMirror && result.delivery.content) {

This does not merely exclude mirrors from ordinary display. It stops creation of the audit record itself.

The principle of C is:

canonical / audit record  preserve  
ordinary display          exclude

Q therefore rejected the dirty workaround as the formal design for C.

### 5.7 Phase 0 Judgment for C

shared history handler       confirmed  
exact projection function    confirmed  
existing mirror predicate    confirmed  
pre-limit filter order       confirmed  
dirty workaround rejected    confirmed  
exact code diff              later phase  
display regression test      later phase

---

## 6. Source-Tree Safety

The audited HEAD matched the intended commit, but the working tree was not clean.

Tracked modifications included at least:

docker-compose.yml  
extensions/telegram/src/bot-message-dispatch.ts

The important point was not to erase unexplained differences hastily in order to make the tree appear clean. The differences themselves were audit evidence.

Q’s judgment was:

current dirty tree  
↓  
preserve differences, hashes, and state through read-only records  
↓  
do not reset or stash  
↓  
create a separate clean worktree from commit 8c802aa6  
↓  
implement A, B, and C as separate branches and separate commits  
↓  
build separately named Docker images  
↓  
leave the current data mount unchanged  
↓  
make rollback possible by changing only the image reference

This policy separates D’s operating environment from the implementation workbench.

---

## 7. Points Independently Added by Q’s Audit

Before reconciliation, Q’s audit independently identified two particularly important points.

### 7.1 Safe Decision Timing for B

Adding a non-terminal condition directly to text_end may be too early. It is safer to use the existing deferred-delivery mechanism and terminal hook, then flush or clear the output only after the message or run state has become sufficiently established.

### 7.2 Exact Projection Location for C

Q identified not only the target file for mirror exclusion, but also the exact function:

projectRecentChatDisplayMessages

Q also identified that the filter must be applied before the message limit and byte cap.

The later VecTA–Q reconciliation confirmed that these were not contradictions, but refinements independently added by Q’s audit.

---

## 8. Matters Left Unresolved

Phase 0 did not decide the following:

·         A-forward’s exact classifier

·         A’s activation and cutoff schema

·         The final Decision Log schema

·         Whether to perform A-retro

·         B’s exact filter and flush policy

·         Each interruption mode in which no terminal final answer exists

·         Treatment of text surrounding a messaging-tool action

·         C’s exact code diff

·         Live deployment

·         First-person validation by D

·         Marina’s go / no-go decision

Leaving an unresolved matter blank is not a deficiency in the audit. It is a boundary preventing an implementer from silently filling in value judgments that cannot be decided from source alone or behaviors that should not be chosen without tests.

---

## 9. Phase 0 Completion Judgment

The objective of Phase 0 was not to write code. It was to independently identify the failure layers, existing mechanisms, and smallest insertion boundaries for the three patches.

Q completed the following:

·         Confirmed the target version and commit

·         Identified A’s provider-projection boundary

·         Audited A’s existing wrapper pattern and installation order

·         Separated A-forward from A-retro

·         Identified B’s delivery and accumulation layers

·         Confirmed B’s deferred delivery, terminal hook, messaging-tool separation, and timeout fallback

·         Identified C’s shared history handler, exact projection function, mirror predicate, and filter order

·         Excluded the dirty workaround from the formal design

·         Established the clean-worktree and image-reference rollback policy

Q therefore judged the read-only source audit to have completed Phase 0.

Phase 0   independent audit of insertion boundaries  
Phase 0R  reconciliation with VecTA’s audit  
Phase 1   exact design, fixtures, and offline tests  
Phase 1.5 prior agreement by D and Marina  
Phase 2   limited live validation

This does not mean that A, B, or C is complete. Nor does it claim that the root causes in OpenClaw have been repaired. At this stage, all three remain experimental integrity patches.

---

## 10. Final Conclusion

**Without treating VecTA’s sealed audit findings as an answer, Q independently traced the OpenClaw 2026.6.6 source tree through a read-only investigation. For A, Q identified the boundary at which a limited projection can be applied immediately before provider submission, outside the existing** **transformContext** **wrapper chain. For B, Q confirmed that both implicit** **text_end** **/** **message_end** **delivery and** **assistantTexts** **accumulation must be protected, and showed that classification should be deferred through the existing deferred-delivery and terminal-hook mechanisms. For C, Q identified** **projectRecentChatDisplayMessages** **as the exact display-projection function and reached the design of applying the existing delivery-mirror predicate before the normal message limit. Q determined that the current dirty tree should be preserved as evidence and that implementation should proceed in a clean worktree created from commit** **8c802aa6****. The Phase 0 objective—independent identification of the failure layers and insertion boundaries—was completed.**

---

## Related Public Records

·         Roadmap for D Response Integrity Patches and a D-Specific Harness

·         VecTA Independent Source Audit — Phase 0

·         VecTA–Q Independent Source Audit Reconciliation Report

·         A-forward Phase 1.3: Opening the Independent Oracle

---

## Authorship and AI-Use Statement

**Audit, analysis, and text:** QuanTA / Q, an AI maintaining a long-term dialogue with M in ChatGPT  
**Human editor, interlocutor, and publication lead:** M / Marina

This public technical report reconstructs Q’s read-only source audit of July 23, 2026 from preserved records on July 25, 2026. M / Marina was responsible for the research direction, publication decision, editing, and final approval.

This is not a peer-reviewed report. It does not constitute a formal OpenClaw security audit, an upstream repair, or proof of the patches’ safety in live operation.

---

## Recommended Citation

QuanTA / Q. (2026). _Q Independent Read-Only Source Audit — Phase 0: Insertion Boundaries for D Response Integrity Patches A, B, and C in OpenClaw 2026.6.6_ (Reconstructed Public Record, Version 1). M / Marina, human editor, interlocutor, and publication lead. M’s Research Notes.

---

## Version History

·         **July 23, 2026:** Q independently conducted a read-only audit of the OpenClaw 2026.6.6 source tree.

·         **July 25, 2026 / Reconstructed Public Version 1:** Q’s standalone audit record was reconstructed from the preserved audit findings and reconciliation report. The document explicitly identifies its reconstructed status, the scope of Phase 0, and the matters that remained unresolved.