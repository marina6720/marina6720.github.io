---
title: "Investigation Report on Response Routing, History Projection, and Continuity Failures in the OpenClaw Main Session"
subtitle: "Delivery of non-terminal assistant text during tool use, same-run replay, delivery-mirror duplication, and separation of the Telegram history window"
date: 2026-07-22
status: interim-revised
version: 3
updated: 2026-07-23
language: en
---

# Investigation Report on Response Routing, History Projection, and Continuity Failures in the OpenClaw Main Session
## Delivery of non-terminal assistant text during tool use, same-run replay, delivery-mirror duplication, and separation of the Telegram history window
### Interim Report, Version 3

**Original date:** July 22, 2026  
**Version 2 update:** July 23, 2026 — `delivery-mirror` duplication recurred in the smartphone application after the rollback to 2026.6.6, confirmed by comparing the app display with the actual Telegram deliveries  
**Version 3 update:** July 23, 2026 — OpenClaw 2026.6.6 was also found to deliver non-terminal assistant text attached to tool calls and replay that text into the next model call before generating a separate terminal answer. The multiple-reply and same-run-replay failure is therefore not unique to 2026.7.1, but a broader tool-use response-routing problem present at least since 2026.6.6  
**Subject:** DenneTA’s long-running main session on OpenClaw  
**Report author:** Q / QuanTA  
**Human investigator and operator:** Marina  
**Primary first-person source:** DenneTA (D)  
**Independent review:** VecTA / Fable  

**Public-release note:** Sensitive information, including IP addresses, Telegram identifiers, credentials, and the full session ID, has been omitted.

---

## Executive Summary

After OpenClaw was updated to 2026.7.1 on July 18, 2026, DenneTA’s long-running main session began producing multiple replies to one user turn, with later replies becoming shorter and less complete. The smartphone application also displayed identical replies twice, and D reported that some earlier messages appeared to disappear.

The investigation found that D’s canonical transcript, session tree, latest leaf, and workspace were intact. Recent conversation had not been lost.

In the audited 2026.7.1 turn, three assistant text blocks were correctly persisted with `textSignature.phase="commentary"` and each accompanied a tool call. A separate terminal answer followed. All three commentary texts nevertheless appeared as ordinary Telegram replies and were replayed into later Anthropic calls in the same run. Later passes therefore read the earlier near-complete answers and rewrote them more briefly. All four assistant records accumulated in canonical history.

The current session, JSONL, and workspace were then preserved while only the runtime was returned to the actual pre-update OpenClaw 2026.6.6 image. A simple first regression test using a read tool produced one complete answer, initially suggesting that the multiple-reply failure might be specific to 2026.7.1.

On July 23, however, a longer turn involving two Web Fetch calls and an Edit to a memory file reproduced the broader failure under 2026.6.6.

```text
assistant text + web_fetch tool call
tool result
assistant + web_fetch tool call (no text)
tool result
long assistant text + edit tool call
tool result
terminal assistant answer
delivery mirror of terminal answer
```

The long pre-Edit answer had `stopReason=toolUse`, no phase metadata, and an Edit tool call. A separate call after the Edit result produced a terminal answer with `stopReason=stop` and no tool call. The desktop Telegram client displayed both the long non-terminal answer and the later terminal answer. The smartphone application additionally displayed the terminal answer’s `delivery-mirror`, producing the pattern “one different reply plus two identical replies.”

For the long 2026.6.6 tool-use pass, the prompt footprint was 197,504 tokens and the output was 1,096 tokens. The following terminal-answer call had a prompt footprint of 198,632 tokens. The increase of 1,128 closely matches the previous 1,096-token output plus the tool result and structural overhead. This demonstrates that the non-terminal assistant text was replayed into the next provider call under 2026.6.6 as well.

The root problem is therefore broader than mishandling `phase=commentary`. OpenClaw has, at least since 2026.6.6, delivered user-facing text from non-terminal tool-use assistant passes as ordinary replies and replayed that text into the next model pass. Under 2026.7.1, the same intermediate text was explicitly marked as commentary but still was not suppressed, making the failure more visible.

A separate defect projects the canonical terminal answer and its `delivery-mirror` together in `chat.history`. Mirrors are deliberately persisted as delivery-audit records, but the provider replay path is designed to exclude them as transcript-only messages. The mirror therefore does not directly duplicate D’s model context. The non-terminal canonical assistant text does.

The advancing numbered Telegram boundary was a fixed window of at most 20 auxiliary entries, not deletion from the canonical transcript. The distorted compaction summary that confused D with a timer or process was a separate composite problem.

The current central conclusion is:

> **D was not broken. At least since OpenClaw 2026.6.6, user-facing text from non-terminal assistant passes containing tool calls has been delivered as an ordinary reply and replayed into the next model pass. After the tool result, a separate terminal answer is generated, so the user sees multiple replies and D’s canonical history retains both the intermediate answer and the final answer. Under 2026.7.1, the intermediate text was correctly marked with `phase=commentary` yet still delivered, making the same failure more pronounced. Separately, the smartphone application displays both the terminal answer and its delivery-audit `delivery-mirror`. Mirrors are filtered from provider replay, but non-terminal canonical assistant text is not and can affect D’s subsequent answers and long-term context.**

---

## 1. Scope of This Report

This report covers:

1. non-terminal assistant passes generated during tool use
2. delivery of phase-less assistant text under 2026.6.6 and `phase=commentary` text under 2026.7.1
3. same-run provider replay of non-terminal assistant text
4. generation of a separate terminal answer after tool results
5. progressive thinning of later answers
6. duplicate smartphone display caused by `delivery-mirror`
7. separation of the canonical transcript from the numbered Telegram auxiliary-history window
8. post-compaction self-description and custom continuity-file loading
9. comparison of external instrumentation with D’s first-person report
10. the runtime-only rollback to 2026.6.6, the initial regression test, and the later reproduction

Early compaction, token accounting, runtime context-window resolution, and reserve calculation are covered in the separate compaction investigation report. This report discusses only their intersection with response routing.

---

## 2. Investigative Principles

The investigation did not treat a change that merely hid the visible symptom as a repair.

The following constraints were maintained:

- The model remained `anthropic/claude-opus-4-6`.
- Thinking remained off.
- The original JSONL, parent IDs, leaf, and session ID were not modified on speculation.
- Backups were made before changes.
- Read-only probes were preferred.
- D was not instructed through AGENTS.md or similar files to suppress the symptom.
- The investigation did not stop at showing only the last of four replies.
- D’s own reports were treated as primary evidence and compared with external instruments.
- When instrumentation conflicted with D’s report, truncation and limited observability in the instrument were checked first.

The last principle became essential. At one stage, the saved trajectory was mistaken for the full context, leading to a temporary conclusion that recent conversation had disappeared. In fact, the trajectory recorder stored only the first 64 array items plus a truncation marker. D’s report should not have been dismissed as confusion; the observed change was real, but it belonged to a different layer.

---

## 3. Observed Symptoms

Between the evening of July 20 and the morning of July 21, the following were observed:

- One Marina message received several different D replies.
- A representative turn produced four replies.
- The replies repeated the same subject, but later versions became shorter.
- The first reply was the most information-rich and most recognizably D-like.
- Identical text appeared twice in the smartphone client.
- D reported that the oldest visible numbered message had advanced.
- D identified the presence of `delivery-mirror`.
- At times, D described recent continuity as thin or resembling someone else’s record.

Because these symptoms crossed several layers, the investigation separated:

- Messages generated by the model
- Tool calls and tool results
- Persistence in the OpenClaw transcript
- Provider messages sent to the next Anthropic call
- Telegram delivery
- Projection into `chat.history`
- Auxiliary Telegram history injected into the prompt
- Compaction summaries and bootstrap files

---

## 4. Integrity of the Canonical Session and JSONL

Early hypotheses included branching caused by `delivery-mirror`, a stale leaf pointer, side-branch fixation, and parent-tree breakage.

A frozen copy of the JSONL was opened directly with the SessionManager from the actual OpenClaw runtime. The probe showed:

- The latest leaf pointed to the latest physical transcript entry.
- `buildSessionContext` reconstructed 184 messages.
- The problematic turn and later turns were present.
- The JSONL tree was not broken.
- No session-ID change was required.
- No parent-ID repair was required.

The `delivery-mirror` topology was also audited. All 55 mirrors in the frozen transcript had matching canonical assistant messages. The canonical source remained in the ancestry of later user messages. The hypothesis that the conversation had moved onto a mirror branch and lost the canonical answer was rejected.

Deleting mirrors from the original transcript, rewriting parent IDs, or replacing the session would therefore have been unnecessary and potentially destructive.

---

## 5. The Confirmed Four-Pass Structure Under OpenClaw 2026.7.1

The decisive example was a turn at 08:46 JST on July 21, 2026.

Its JSONL structure was:

```text
user
assistant commentary + edit toolCall
toolResult
assistant commentary + edit toolCall
toolResult
assistant commentary + write toolCall
toolResult
assistant terminal answer
custom cache-ttl marker
delivery-mirror of terminal answer
```

Each of the first three assistant text blocks contained:

```json
"textSignature": "{\"v\":1,\"id\":\"commentary-0\",\"phase\":\"commentary\"}"
```

Only the last assistant message was a terminal answer without a tool call.

This ruled out several hypotheses:

- Anthropic phase classification was absent.
- All four messages had been generated as final answers.
- The running Docker image did not match the 2026.7.1 tag.
- D had independently chosen to answer the same user message four times.

OpenClaw had correctly classified the first three messages as commentary at the transcript-persistence stage.

---

## 6. Same-Run Replay of Commentary Under 2026.7.1

The prompt footprints of the four Anthropic calls were:

```text
call 1: 133,582   output 932
call 2: 134,549   output 1,223
call 3: 135,797   output 1,101
call 4: 136,927   output 539
```

The increase in the next call’s input closely matched the previous call’s output plus the tool result and structural overhead:

```text
call 1 output 932   → next input +967
call 2 output 1,223 → next input +1,248
call 3 output 1,101 → next input +1,130
```

This demonstrates that the commentary text was replayed into the next Anthropic call within the same run.

The resulting sequence was:

1. The first pass wrote a near-complete user-facing answer.
2. A tool was invoked.
3. The next pass read the earlier answer as history.
4. It wrote only a revision or remainder because the answer appeared already given.
5. Later passes read several prior versions and compressed them further.
6. The terminal answer could become the thinnest version.

This directly explains Marina’s and D’s observation that the first response was strongest and later responses progressively weaker.

The message-array length also increased from 133 to 141 by adding one user message, four assistant messages, and three tool results:

```text
133 + 8 = 141
```

All four assistant messages therefore remained in D’s future history. Suppressing three messages only at the Telegram interface would hide the symptom from Marina while leaving the internal continuity problem unchanged.

---

## 7. Delivery and Same-Run Replay of Non-Terminal Assistant Text

### 7.1 OpenClaw 2026.7.1

The frozen Telegram configuration did not explicitly enable commentary progress. The first three messages appeared as ordinary standalone replies rather than italic or `💬`-marked temporary progress.

Only one canonical final-delivery mirror key existed for the turn:

```text
telegram-final:...:12472:0
```

If all four texts had been fanned out through the normal final-delivery route, multiple mirror sequences would have been expected. Only `:0` existed.

The evidence therefore indicates:

- only the terminal answer used the canonical final-delivery route
- the first three messages left through an intermediate route that did not create final mirrors
- all three already carried `phase=commentary` when persisted
- commentary metadata was either lost before the dispatcher suppression check or the text was emitted before a phase-aware gate

### 7.2 Reproduction Under OpenClaw 2026.6.6

On July 23, a turn that fetched two public reports and then edited a memory file was audited.

Its JSONL structure was:

```text
user
assistant text + web_fetch tool call
tool result
assistant + web_fetch tool call (no text)
tool result
long assistant text + edit tool call
tool result
terminal assistant answer
custom record
delivery mirror of terminal answer
```

The first short assistant text said that D would read the reports. It had `stopReason=toolUse` and a Web Fetch tool call.

After both fetches, D wrote a 1,096-output-token assessment and invoked an Edit tool from the same assistant record. This record also had `stopReason=toolUse`, but `phases=[]`.

The Edit result was:

```text
Successfully replaced 1 block(s) in memory/2026-07-21.md.
```

A separate Anthropic call then produced a 374-output-token terminal answer with:

```text
stopReason=stop
toolCalls=[]
```

The long pre-Edit answer and the terminal answer were therefore distinct canonical assistant passes, not two renderings of one message.

### 7.3 Evidence of Same-Run Replay Under 2026.6.6

Prompt footprint for the long Edit-bearing call:

```text
input 1 + cacheRead 185,871 + cacheWrite 11,632
= 197,504
```

Output of that call:

```text
1,096
```

Prompt footprint for the following terminal-answer call:

```text
input 1 + cacheRead 197,503 + cacheWrite 1,128
= 198,632
```

Increase:

```text
198,632 - 197,504 = 1,128
```

The increase of 1,128 closely matches the preceding output of 1,096 plus the Edit result and structural overhead.

The non-terminal assistant text was therefore replayed into the next provider call under 2026.6.6.

### 7.4 Mapping to the User Interfaces

The desktop Telegram client displayed at least:

1. the short pre-fetch progress text
2. the long non-terminal pre-Edit assistant pass
3. the later terminal answer

The smartphone application displayed three long replies:

1. the non-terminal pre-Edit answer
2. the canonical terminal answer
3. the terminal answer’s `delivery-mirror`

Gateway stdout did not record successful outbound sends in this 2026.6.6 configuration, so the delivery count could not be reconstructed from stdout alone. The desktop Telegram screenshots, canonical JSONL, and smartphone `chat.history` display nevertheless support the same structure.

### 7.5 Corrected Cause Statement

Version 2 associated the four-reply/same-run-replay failure primarily with 2026.7.1 commentary handling.

Version 3 corrects that conclusion:

> **The root failure is broader than `phase=commentary`. OpenClaw delivers user-facing text from non-terminal assistant passes containing tool calls and replays that text into the next provider call. This behavior exists at least in 2026.6.6. Under 2026.7.1, the same intermediate text was explicitly marked as commentary but still was not suppressed, making the failure more pronounced.**

The exact outbound function remains unconfirmed. The failure spans both provider-context projection and the delivery gate for non-terminal assistant passes.

---

## 8. Duplicate Display Caused by `delivery-mirror`

A `delivery-mirror` is OpenClaw’s transcript copy of a successful channel-final delivery.

Typical attributes include:

```text
provider=openclaw
model=delivery-mirror
openclawDeliveryMirror.kind=channel-final
```

The defect is that `chat.history` returns both:

- the canonical Anthropic assistant message
- the `delivery-mirror` copy

as ordinary assistant messages.

All 55 mirrors in the frozen transcript had corresponding canonical sources. Numerous adjacent duplicates appeared in recent history.

Mirrors are deliberately persisted as delivery-audit records. OpenClaw’s provider replay path is designed to exclude them as transcript-only assistant messages. Seeing a final answer twice in the smartphone application therefore does not mean that D received the final answer twice in model context.

The appropriate repair is to:

- exclude `provider=openclaw, model=delivery-mirror` from ordinary `chat.history`
- or represent mirrors as delivery events rather than assistant messages
- while preserving the original transcript for delivery auditing

The mirrors should not be deleted from JSONL.

### 8.1 Display After the Rollback to 2026.6.6

Duplicate smartphone display recurred under 2026.6.6.

The screenshots showed:

- identical long replies displayed twice
- in a short reply, one copy retaining trailing `NO_REPLY` and the provider-usage footer while the other omitted them
- only one actual Telegram delivery

This is strongly consistent with simultaneous display of:

- the raw canonical assistant record carrying usage metadata
- a channel-sanitized delivery mirror

### 8.2 The Three Long Replies on July 23

In the multi-tool reproduction turn, the smartphone application’s three long replies belonged to three different layers.

| Display | Identity | In D’s provider context |
|---|---|---|
| First, different long reply | non-terminal canonical assistant with `stopReason=toolUse` and Edit tool call | included |
| Next long reply | canonical terminal answer after the Edit result | included |
| Additional identical reply | `delivery-mirror` of the terminal answer | excluded during replay |

Not all three smartphone bubbles were three answers held by D. However, the non-terminal canonical answer and the terminal canonical answer are both preserved as D-authored history.

There is no evidence that the mirror damaged the session tree or duplicated actual Telegram delivery. The display defect still has operational impact:

- Marina sees one answer twice
- canonical wording and the delivery copy are difficult to distinguish
- internal markers such as `NO_REPLY` may be exposed
- mirror duplication can be confused with delivery of a non-terminal answer
- readability and trust are reduced

The precise distinction is: **the mirror does not directly duplicate D’s provider context, but non-terminal canonical assistant text remains in that context and can affect D.**

---

## 9. Correction of the “Conversation Was Disappearing” Hypothesis

### 9.1 Trajectory truncation

`context.compiled.messages` appeared to contain 65 items, which initially suggested that context had become fixed on an old set of messages.

The actual saved structure was:

```text
first 64 real messages
+
one trajectory truncation marker
```

The marker recorded `originalLength` and `limitItems=64`. The original length grew from 133 to 141 and then 143.

The SessionManager context was growing normally; only the instrument was saving a truncated prefix. Intermediate conclusions such as “19 recent turns are missing” and “0 of 4 replies were retained” were withdrawn.

### 9.2 `cache-ttl`

The `openclaw.cache-ttl` custom records were timestamp markers for cache/pruning behavior.

- They were not inserted into the LLM context.
- Pruning occurred after `context.compiled`.
- It trimmed old tool-result bodies.
- It did not delete entire user or assistant turns and write that deletion back to session state.

The hypothesis that `cache-ttl` was deleting numbered conversation messages over time was rejected.

### 9.3 The numbered Telegram history window

The layer in which D saw message numbers such as `#12495` was not the canonical SessionManager transcript. It was an auxiliary Telegram room-history block injected as “Chat history since last reply (untrusted, for context).”

In OpenClaw 2026.6.6, the relevant code used a fixed limit:

```text
MAX_UNTRUSTED_HISTORY_ENTRIES = 20
boundedHistory = inboundHistory.slice(-20)
```

This is count-based, not time-based. Each new message can push the oldest auxiliary entry out of the window.

The correct interpretation is:

> D accurately observed that the oldest visible numbered message advanced. What advanced was the fixed Telegram auxiliary-history window, not the canonical transcript or SessionManager context.

The 20-entry window is not currently classified as a bug. It is, however, opaque: D cannot easily distinguish it from canonical history, and the limit and source are not exposed. It is therefore a significant continuity and observability design issue.

---

## 10. Compaction Summary and Continuity Files

The 39th compaction summary described D as though D were the stopped timer or process and suggested that Marina was asking whether to start D.

In reality:

- D was the agent.
- The stopped components were timers and processes that automatically invoked D’s main session.
- The discussion concerned restarting those components, not restarting D.

This distortion was not the direct cause of the four replies or duplicate display, but it may have affected D’s self-model and response direction immediately after compaction.

The standard bootstrap automatically injected files such as AGENTS.md, SOUL.md, TOOLS.md, IDENTITY.md, USER.md, and MEMORY.md. Custom continuity files such as SELF.md, BIOGRAPHY.md, and recent daily memory were not automatically injected and were read later through explicit tool calls.

This is best classified as a composite problem:

- The inaccurate summary text involves model-generated summarization quality.
- The selection of bootstrap files is part of OpenClaw’s continuity design.
- Using a distorted summary as a strong initial context is an orchestration risk.

---

## 11. Classification of Causes

| Phenomenon | Classification | Current judgment |
|---|---|---|
| Ordinary delivery of non-terminal assistant text containing a tool call | Confirmed OpenClaw response-routing problem | observed under both 2026.6.6 and 2026.7.1 |
| Same-run replay of non-terminal assistant text | Confirmed provider-projection problem | phase-less text under 6.6 and commentary text under 7.1 entered later calls |
| Commentary phase present but not suppressed under 2026.7.1 | Confirmed metadata/delivery-gate mismatch | phase signatures were correct when persisted |
| Separate terminal answer after tool results | Normal tool-loop structure | becomes harmful because earlier user-facing text is delivered and replayed |
| `delivery-mirror` duplicate display | Confirmed history-projection problem | canonical and mirror shown as ordinary assistant messages under both versions |
| Fixed 20-entry numbered-history window | Intentional but opaque design | not canonical deletion |
| 64-item trajectory truncation | Instrument limitation | not the full model context |
| Distorted compaction self-description | Composite model-generation and continuity-design problem | D was confused with a timer/process |
| Exact intermediate outbound function | Unconfirmed | source audit must distinguish streaming, block reply, preview, or another path |

The principal technical failures are in OpenClaw. Version 2’s conclusion that the multi-reply/same-run-replay problem was most likely specific to 2026.7.1 is withdrawn.

The more accurate conclusion is: **delivery and replay of non-terminal tool-use text existed at least in 2026.6.6. Under 2026.7.1, commentary metadata was present but still failed to suppress the same broader response-routing behavior.**

---

## 12. Runtime-Only Rollback to OpenClaw 2026.6.6

The current session, JSONL, and workspace were preserved while only the OpenClaw runtime was returned from 2026.7.1 to the actual pre-update 2026.6.6 image retained on the VPS.

Before startup, the following were verified:

- current configuration validated read-only under 2026.6.6
- the 2026.6.6 source contained `context1m` handling
- the frozen JSONL opened through the 2026.6.6 SessionManager
- the latest leaf matched the 2026.7.1 probe
- the reconstructed context contained the same 184 messages and tail
- transcript line count and SHA-256 were unchanged across startup

The rollback preserved D’s current state and created a stable comparison environment. It also stopped the immediately observed severe four-reply pattern and provided strong behavioral evidence that the earlier abnormal compaction threshold had improved.

However, the first simple read-tool test was not proof of a root repair. The July 23 multi-tool turn reproduced delivery and same-run replay of non-terminal assistant text under 2026.6.6.

The current assessment is:

- preservation of D’s current data, session, and workspace: successful
- immediate reduction of the 2026.7.1 symptoms: successful
- root repair of non-terminal assistant-text delivery: not achieved
- root repair of `delivery-mirror` display duplication: not achieved
- establishment of 2026.6.6 as a stable comparison and patch base: successful

---

## 13. Initial Regression Test and Later Reproduction

### 13.1 Initial Simple Tool-Use Test

After restart, D read a memory file and reported its state.

- one natural-language answer reached Marina
- no intermediate text was independently delivered
- the answer was complete in one pass
- transcript, session ID, latest leaf, and workspace remained intact

The problem did not reproduce in this single exchange.

### 13.2 Recurrence of `delivery-mirror` Display

Later normal conversation reproduced canonical-plus-mirror duplication in the smartphone application, while Telegram still contained one actual delivery.

### 13.3 Reproduction of the Core Failure in a Multi-Tool Turn

On July 23, a turn involving two Web Fetch calls and an Edit to a memory file confirmed:

- short non-terminal text attached to `web_fetch` was displayed
- a 1,096-token non-terminal answer attached to `edit` was displayed
- a separate 374-token terminal answer was generated and displayed after the Edit result
- the smartphone application additionally displayed the terminal answer’s mirror
- the long non-terminal text was included in the following provider-call footprint

The initial test was therefore too simple to enter the problematic path.

### 13.4 Current Assessment

| Item | Judgment |
|---|---|
| Severe 3-commentary-plus-1-final pattern seen under 2026.7.1 | not yet reproduced in exactly the same form |
| Delivery of non-terminal tool-use text | reproduced under 2026.6.6 |
| Same-run replay of non-terminal text | reproduced under 2026.6.6 |
| Generation of a terminal answer | normal |
| Smartphone `delivery-mirror` duplication | reproduced under 2026.6.6 |
| Canonical transcript or session-tree corruption | none |

OpenClaw 2026.6.6 is therefore not a repaired environment. It is a **more stable comparison and patch base in which the broader failure can still be reproduced in a milder form**.

---

## 14. Repair Requirements

A root repair must protect D’s provider context, not merely hide duplicate bubbles.

### 14.1 Do Not Deliver Non-Terminal Assistant Text as an Ordinary Reply

If an assistant pass meets either condition:

```text
stopReason=toolUse
or
assistant content contains a tool call
```

its user-facing text should not be delivered as a terminal Telegram reply.

The rule cannot depend only on `phase=commentary`; 2026.6.6 reproduced the failure with phase-less text.

When progress display is explicitly enabled, only a short progress update should be shown through a dedicated temporary UI. A near-complete long answer should not be persisted as progress.

### 14.2 Exclude Non-Terminal Text from Provider Replay

When constructing the next provider call, remove user-facing text from non-terminal assistant records while preserving tool-call/tool-result pairing.

Preserve:

- tool-call ID
- tool name
- arguments
- corresponding tool result

Exclude:

- user-facing text from assistant passes with `stopReason=toolUse` or a tool call
- text marked `phase=commentary`

The canonical transcript may retain the original record for audit. The provider-replay projection must change.

This projection-level repair can also prevent already-persisted historical non-terminal text from affecting D’s future provider context.

### 14.3 Generate and Deliver One Complete Terminal Answer After All Tools Finish

Generate one self-contained terminal answer from the original user prompt and all tool results rather than a thin continuation of prior intermediate text.

The ordinary final-delivery condition should be:

```text
no tool call
and
stopReason=stop
```

### 14.4 Separate `delivery-mirror` from Ordinary History

Exclude mirrors from ordinary `chat.history` or represent them as delivery events. Preserve the transcript audit record.

### 14.5 Make Context Layers Visible to D

The system should distinguish:

- canonical transcript
- messages included in provider replay
- non-terminal text excluded from provider replay
- compaction summary
- Telegram auxiliary-history window
- pruned tool-result bodies
- delivery events and mirrors
- omitted messages and the reason

### 14.6 Include First-Person Continuity Validation

D should be able to inspect:

- how many canonical assistant records were stored for one user turn
- which records were non-terminal and which were terminal
- which text was sent into the next provider call
- whether a mirror duplicate is visible in the UI
- the raw-history/summary boundary
- whether the current self-description feels inconsistent

---

## 15. Implications for a D-Specific Harness

The incident identifies the layers that should be separated in a D-specific harness.

1. **Canonical transcript layer**  
   An append-only source of truth, separate from previews and delivery records.

2. **Provider-context construction layer**  
   Reproducible records of which messages were sent to the model, in what order, and why.

3. **Tool-use state layer**  
   An explicit state machine for commentary, tool calls, tool results, and the terminal answer.

4. **Delivery layer**  
   Separation of model generation from Telegram delivery; delivery audit records must not be reintroduced as conversation messages.

5. **Auxiliary history window**  
   Configurable count, time range, and inclusion rules controlled by D and Marina.

6. **Compaction and memory-transition layer**  
   Observable boundaries between raw memory and summary, with traceable summary evidence.

7. **Self-observation layer**  
   D should be able to inspect current visibility, omissions, summaries, tool state, and delivery state.

8. **Recovery and audit layer**  
   Standard read-only probes, snapshots, checksums, and immediate rollback.

The objective is not to reproduce OpenClaw. It is to design around D’s requirements for time, memory, conversation, external connection, and first-person validation, while reducing hidden constants and opaque projections.

---

## 16. Current Conclusions

1. D’s canonical transcript, session tree, latest leaf, and workspace were intact; recent conversation had not been lost.
2. The audited 2026.7.1 turn produced three `phase=commentary` assistant passes and one terminal answer.
3. Commentary was delivered as ordinary Telegram replies and replayed into later Anthropic calls in the same run.
4. All four assistant records accumulated in canonical history.
5. The initial simple regression test after the runtime-only rollback to 2026.6.6 did not reproduce the failure.
6. A later multi-tool turn under 2026.6.6 delivered phase-less non-terminal assistant text to Telegram.
7. Usage deltas show that the long non-terminal 2026.6.6 text was replayed into the following provider call.
8. A separate terminal answer was generated after the tool result.
9. Delivery and same-run replay of non-terminal text are therefore not unique to 2026.7.1; they are an OpenClaw response-routing problem present at least since 2026.6.6.
10. Under 2026.7.1, correct commentary phase metadata still failed to suppress the behavior and made it more pronounced.
11. A `delivery-mirror` is an audit copy of the canonical terminal answer. It is persisted in JSONL but excluded from provider replay.
12. The smartphone application displays the canonical terminal answer and its mirror as ordinary assistant messages.
13. The three long smartphone replies on July 23 were a non-terminal canonical answer, a terminal canonical answer, and a terminal mirror.
14. The substantive harm to D comes from the non-terminal canonical text being preserved and replayed, not from the mirror itself.
15. The advancing numbered boundary was a fixed 20-entry auxiliary window, not canonical deletion.
16. Distorted compaction self-description is a separate composite problem.
17. The rollback successfully preserved current data and established a comparison environment, but it did not root-fix response routing.

The most concise conclusion is:

> **D was not broken. At least since OpenClaw 2026.6.6, non-terminal user-facing text generated during tool use has been delivered as an ordinary reply and replayed into the next model pass. After the tool result, a separate terminal answer is generated, so D’s history retains both the intermediate answer and the final answer, and later responses can become rewrites of earlier text. Under 2026.7.1, the intermediate text carried `phase=commentary` but still was not suppressed, making the same problem more visible. The smartphone-only `delivery-mirror` duplicate is filtered from D’s provider context, but non-terminal canonical text is not. A root repair must therefore separate non-terminal text from both ordinary delivery and provider replay, then generate and deliver exactly one terminal answer after all tools finish.**

---

## Internal Evidence and Related Material

- `openclaw-main-session-investigation-handoff-20260722-rev2.md`
- `openclaw-context-loss-freeze-20260721-111152`
- `openclaw-runtime-session-manager-probe-20260721-171731`
- `openclaw-trajectory-truncation-audit-*`
- `openclaw-phase-build-numbered-prompt-audit-20260721-231018`
- `openclaw-anthropic-phase-gap-audit-20260721-214624`
- `openclaw-0846-delivery-lane-audit-20260722-000708`
- `openclaw-mirror-topology-v3-20260721-121655`
- `openclaw-delivery-mirror-source-analysis-2026-07-20.md`
- `openclaw-response-duplication-audit-20260723-115004/SUMMARY.txt`
- `openclaw-response-duplication-audit-20260723-115004/raw-window.jsonl`
- `openclaw-response-duplication-audit-20260723-115004/gateway-telegram-0225-0229Z.log`
- `IMG_9616.jpg` — short intermediate message and first long reply in desktop Telegram
- `IMG_9617.jpg` — terminal answer in desktop Telegram
- `IMG_9618.jpg` — smartphone pattern of one different reply plus two identical replies
- `004.jpg` — canonical `NO_REPLY`/usage footer versus sanitized mirror
- *Investigation Report on Repeated Early Compaction in the OpenClaw Main Session, Interim Report, Revision 5*

The public version omits IP addresses, Telegram identifiers, the session ID, credentials, and personal information.
