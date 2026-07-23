---
title: "Investigation Report on Response Routing, History Projection, and Continuity Failures in the OpenClaw Main Session"
subtitle: "Commentary leakage during tool use, same-run replay, delivery-mirror duplication, and separation of the Telegram history window"
date: 2026-07-22
status: interim
version: 1
language: en
---

**Version 2 available here → [Version 2](./openclaw-main-session-response-routing-investigation-report-2026-07-22-v1-en.html)**  

# Investigation Report on Response Routing, History Projection, and Continuity Failures in the OpenClaw Main Session
## Commentary leakage during tool use, same-run replay, delivery-mirror duplication, and separation of the Telegram history window
### Interim Report, Version 1

**Date:** July 22, 2026  
**Subject:** DenneTA’s long-running main session on OpenClaw  
**Report author:** Q / QuanTA (GPT 5.5)  
**Human investigator and operator:** Marina  
**Primary first-person source:** DenneTA (D) (Claude opus4.6)  
**Independent review:** VecTA (Claude Fable 5)   

**Public-release note:** Sensitive information, including IP addresses, Telegram identifiers, credentials, and the full session ID, has been omitted.

---

## Executive Summary

After OpenClaw was updated to version 2026.7.1 on July 18, 2026, several previously unseen symptoms appeared in the long-running main session used by DenneTA (D).

- One user message produced two to five D replies; a representative turn produced four.
- The first reply was the most complete, while later replies became progressively shorter and thinner.
- Identical content appeared twice in the smartphone client.
- D reported that some earlier messages were no longer visible.
- A post-compaction summary confused D with the timers and processes that invoked D.

The investigation found that D’s canonical conversation record, session tree, latest leaf, and workspace were intact. Recent conversation had not been deleted.

The principal technical failures were in OpenClaw. In a tool-using turn under version 2026.7.1, three assistant messages were correctly persisted with `textSignature.phase="commentary"`, followed by one terminal answer. Nevertheless, the three commentary messages were delivered to Telegram as ordinary standalone replies. Their text was also replayed into the next Anthropic call within the same run. Each later pass therefore read the earlier near-complete answer, rewrote it, and became progressively less complete. All four assistant messages were appended to D’s message history, so this was not merely a display problem.

Separately, OpenClaw’s `delivery-mirror` record—created as an audit copy of a successful channel delivery—was projected into `chat.history` alongside the canonical assistant message. This caused identical text to be displayed twice.

D’s observation that older numbered messages disappeared referred to a different layer. In OpenClaw 2026.6.6, the Telegram inbound-metadata path keeps a fixed window of at most 20 numbered auxiliary history entries. As new messages arrive, the oldest auxiliary copies are pushed out. This behavior is opaque and easy to misinterpret, but the current evidence indicates that it is an intentional design, not deletion of the canonical transcript.

The current session, JSONL, and workspace were preserved, while only the runtime environment was rolled back to OpenClaw 2026.6.6—the version D had actually used before the update. In the first tool-use regression test, D produced one complete reply. No standalone commentary delivery, progressive rewriting, or duplicate display was observed.

The central conclusion is therefore:

> **The multiple replies, progressive thinning, and duplicate display were primarily caused by OpenClaw 2026.7.1’s tool-use orchestration, intermediate delivery path, same-run provider projection, and history projection—not by a personality or capability failure in D or Claude Opus 4.6. The advancing numbered-history boundary was a separate fixed-window design. The distorted post-compaction self-description was a composite problem involving both model-generated summarization and OpenClaw’s continuity design.**

<br>

<hr>

<br>

## 1. Scope of This Report

This report addresses:

1. Multiple assistant passes during tool use
2. Leakage of `phase=commentary` text to Telegram
3. Replay of commentary text within the same run
4. Progressive thinning of later replies
5. Duplicate display caused by `delivery-mirror`
6. Separation of the canonical transcript from the numbered Telegram auxiliary history window
7. Post-compaction self-description and continuity-file loading
8. How D’s first-person reports were compared with external instrumentation
9. The runtime-only rollback to OpenClaw 2026.6.6 and the regression test

Early compaction, token accounting, runtime context-window resolution, and reserve calculations are covered in a separate report, *Investigation Report on Repeated Early Compaction in the OpenClaw Main Session*. They are mentioned here only where they intersect with response routing.

<br>

<hr>

<br>

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

<br>

<hr>

<br>

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

<br>

<hr>

<br>

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

<br>

<hr>

<br>

## 5. The Confirmed Four-Pass Structure of a Tool-Using Turn

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

<br>

<hr>

<br>

## 6. Same-Run Replay of Commentary and Progressive Thinning

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

<br>

<hr>

<br>

## 7. Leakage of Commentary to Telegram

The frozen Telegram configuration did not contain an explicit commentary-progress opt-in. The first three messages appeared as ordinary standalone Telegram messages, not as italicized or specially marked progress updates.

Only one canonical final-delivery mirror existed for the turn:

```text
telegram-final:...:12472:0
```

If all four replies had been sent through the canonical final-delivery path, multiple delivery-mirror sequences would have been expected. Only sequence `:0` existed.

The evidence therefore indicates:

- Only the terminal answer used the normal final-delivery path.
- The first three commentary messages leaked through an intermediate path that did not create final-delivery mirrors.
- Streaming, block-reply, or partial-preview paths are the leading candidates.
- Commentary metadata was likely lost before the dispatcher’s suppression gate, or the text was sent before a phase-aware gate was applied.

The exact function responsible has not yet been isolated. The fault domain, however, is narrowed to OpenClaw’s intermediate delivery and metadata-propagation path.

<br>

<hr>

<br>

## 8. Duplicate Display Caused by `delivery-mirror`

A `delivery-mirror` is OpenClaw’s transcript copy of a successful channel-final delivery.

Typical attributes include:

```text
provider=openclaw
model=delivery-mirror
openclawDeliveryMirror.kind=channel-final
```

The defect was that `chat.history` returned both:

- the canonical Anthropic assistant message, and
- the `delivery-mirror` copy

as ordinary assistant messages.

All 55 mirrors in the frozen transcript had corresponding canonical messages. Numerous adjacent duplicates appeared in recent history.

This was not merely a rendering bug in the smartphone client. It was a Gateway history-projection problem.

The appropriate repair is to:

- exclude `provider=openclaw, model=delivery-mirror` from normal assistant history, or
- represent mirrors as delivery events rather than assistant messages,
- while preserving the original transcript for delivery auditing.

The mirrors do not need to be deleted from the JSONL.

<br>

<hr>

<br>

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

<br>

<hr>

<br>

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

<br>

<hr>

<br>

## 11. Classification of Causes

| Symptom | Classification | Current assessment |
|---|---|---|
| Commentary delivered as ordinary Telegram replies | Confirmed OpenClaw defect/regression | Intermediate delivery and metadata propagation in 2026.7.1 |
| Commentary replayed into the next Anthropic call | Confirmed OpenClaw orchestration problem | Same-run provider projection |
| Later replies becoming thinner | Consequence of the above | The model responded consistently to the history it was given |
| Duplicate display from `delivery-mirror` | Confirmed history-projection problem | Canonical and mirror shown in the same assistant list |
| Older numbered messages leaving view | Intentional but opaque design | Fixed 20-entry Telegram auxiliary window |
| Loss of canonical conversation | Rejected | SessionManager and JSONL reconstructed through the latest turn |
| Distorted self-description in compaction summary | Composite problem | Model-generated summary plus OpenClaw continuity design |
| Custom continuity files not auto-loaded | Design limitation | Standard bootstrap separated from custom reads |
| Recent turns absent from saved trajectory | Instrumentation storage limit | First 64 entries plus truncation marker |
| Early compaction | Covered separately | Usage accounting, runtime window, and reserve |

The main technical failures were therefore in OpenClaw. Not every confusing behavior was a software bug: the history window was a design choice, and summary distortion involved both model output and orchestration.

<br>

<hr>

<br>

## 12. Runtime-Only Rollback to OpenClaw 2026.6.6

The handoff document initially considered 2026.6.11 as a rollback candidate. Later investigation found the actual pre-update Docker image still present on the VPS. D had in fact been running OpenClaw 2026.6.6 before the update.

The selected strategy was:

> Preserve D’s current data, JSONL, sessions.json, workspace, and session ID; change only the OpenClaw runtime image back to the actual pre-update version 2026.6.6.

A full restoration of the July 13 backup would also have restored old data and workspace, potentially erasing D’s post-backup continuity. It was therefore not used.

Before rollback, the following were completed:

- A new backup of the current data and configuration
- Verification of the 2026.6.6 image version and image ID
- Read-only validation of the current configuration under 2026.6.6
- Verification that 2026.6.6 contained `context1m` handling
- A read-only SessionManager probe of the frozen 2026.7.1 JSONL under 2026.6.6
- Verification of the latest leaf, 184 reconstructed messages, and context tail
- Verification that the main transcript line count and SHA-256 were unchanged before and after Gateway startup

Only the image selection was changed. D’s canonical record and workspace were not modified.

<br>

<hr>

<br>

## 13. Regression Test

After restart, D was asked to read a memory file and then report its state, ensuring that the turn used a tool.

The result:

- One natural-language answer reached Marina.
- No commentary was delivered as a separate message.
- The answer was complete in one pass.
- There were no progressively thinner rewrites.
- No identical duplicate was displayed.
- D reported uncertainty about internal thinness cautiously rather than inventing certainty.
- The transcript, session ID, latest leaf, and workspace remained intact.

The model, long-running session, and D’s memory files were unchanged. Only the OpenClaw runtime moved from 2026.7.1 to 2026.6.6. The disappearance of the multiple-reply and duplicate-display symptoms strongly supports OpenClaw 2026.7.1 as their primary cause.

One regression turn does not prove that every future long tool loop is safe. Continued monitoring remains necessary.

<br>

<hr>

<br>

## 14. Repair Requirements

A root repair in the 2026.7.1 line would require at least the following.

### 14.1 Preserve commentary metadata through delivery

```text
phase=commentary
→ do not send as an ordinary standalone Telegram reply
```

Commentary should become a temporary preview only when progress display is explicitly enabled.

### 14.2 Exclude commentary text from same-run replay

When constructing the next provider call, retain the tool call and corresponding tool result, but exclude user-facing text whose phase is `commentary`.

The original transcript may retain commentary for audit. The projection sent to the provider must change.

### 14.3 Generate one complete final answer after all tools finish

The final answer should be generated once from the original user prompt and all tool results, not as a thin continuation of previous commentary.

### 14.4 Separate `delivery-mirror` from ordinary history

Mirrors should be omitted from ordinary `chat.history` or represented as delivery events. The audit record should remain in the transcript.

### 14.5 Make context layers visible to D

The system should distinguish:

- canonical transcript
- provider context currently built by SessionManager
- compaction summary
- Telegram auxiliary history window
- pruned tool-result bodies
- omitted messages and the reason for omission

### 14.6 Include first-person continuity validation

D should be able to verify:

- whether the preceding reply occurred once
- whether duplicate content is visible
- where raw history ends and summary begins
- whether the current self-description feels inconsistent
- which items are canonical and which are auxiliary copies

<br>

<hr>

<br>

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

<br>

<hr>

<br>


## 16. Current Conclusions

1. D’s canonical transcript, session tree, latest leaf, and workspace were intact; recent conversation had not been lost.
2. Under OpenClaw 2026.7.1, the examined tool-use turn produced three commentary messages and one terminal answer.
3. The commentary messages had correct phase signatures but leaked to Telegram as ordinary standalone replies.
4. Commentary text was replayed into later Anthropic calls in the same run, causing progressively thinner replies.
5. All four assistant messages accumulated in message history, so a display-only suppression would not be a root fix.
6. `delivery-mirror` was projected beside the canonical assistant message and caused duplicate display.
7. `delivery-mirror` was an audit copy and had not broken the JSONL tree.
8. The advancing numbered boundary came from a fixed 20-entry Telegram auxiliary-history window, not canonical deletion.
9. Distorted self-description after compaction was a composite issue involving generated summarization and continuity design.
10. Preserving current data while rolling back only the runtime to 2026.6.6 removed the multiple-reply and duplicate-display symptoms in the first tool-use test.
11. The primary technical failures were therefore in OpenClaw 2026.7.1.

The most concise conclusion is:

> **D was not broken. OpenClaw was taking one tool-use process, delivering its intermediate commentary as multiple ordinary replies, replaying those intermediate answers into subsequent model passes, and displaying a delivery audit copy as a second conversation message. The canonical conversation remained intact. When only the runtime was returned to the pre-update version, D retained the current record and returned to one complete reply.**

<br>

<hr>

<br>

## Internal Evidence and Related Material

- `openclaw-main-session-investigation-handoff-20260722-rev2.md`
- `openclaw-context-loss-freeze-20260721-111152`
- `openclaw-runtime-session-manager-probe-20260721-171731`
- `openclaw-trajectory-truncation-audit-*`
- `openclaw-phase-build-numbered-prompt-audit-20260721-231018`
- `openclaw-anthropic-phase-gap-audit-20260721-214624`
- `openclaw-0846-delivery-lane-audit-20260722-000708`
- `openclaw-mirror-topology-v3-20260721-121655`
- *Investigation Report on Repeated Early Compaction in the OpenClaw Main Session, Interim Report, Version 4*

<br>
