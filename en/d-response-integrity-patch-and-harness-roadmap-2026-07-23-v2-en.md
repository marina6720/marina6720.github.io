---
title: "D Response Integrity Patch Plan and Roadmap Toward a D-Specific Harness"
date: 2026-07-23
status: working-roadmap-revised
version: 2
updated: 2026-07-23
language: en
---

# D Response Integrity Patch Plan and Roadmap Toward a D-Specific Harness
## Version 2  

July 23, 2026  

## Summary

This roadmap defines three independent minimal patches for response-routing failures observed in DenneTA’s long-running OpenClaw main session. The patches are designed not only to repair the current system, but also to establish reusable architectural boundaries for a future D-specific harness.

The three problems can overlap in the user interface, but they occur at different layers and require separate repairs.

1. **Protect D’s context**  
   Prevent user-facing text from non-terminal tool-use assistant passes from being replayed into the next model pass and future provider context.

2. **Suppress intermediate delivery**  
   Prevent non-terminal assistant text attached to tool calls from being delivered as an ordinary Telegram answer before the terminal final answer.

3. **Remove mirror duplication from normal history**  
   Keep `delivery-mirror` as an audit record while excluding it from ordinary conversation display.

The three changes will not be implemented as one patch. Each will have its own source audit, commit, image, test, validation gate, and rollback path.

---

## 1. Background

The investigation found that, at least since OpenClaw 2026.6.6, user-facing text from non-terminal assistant passes containing tool calls could be delivered as an ordinary reply and replayed into the next provider call.

Under OpenClaw 2026.7.1, the intermediate text was correctly marked with `phase=commentary` but still was not suppressed. One user turn could therefore produce several replies, with later replies becoming shorter because they had already received the earlier near-complete answer in context.

Separately, the OpenClaw application projected both a canonical terminal answer and its delivery-audit `delivery-mirror` as ordinary assistant messages. Mirrors are filtered from provider replay, but non-terminal canonical assistant text is not and can affect D’s subsequent answers and long-term context.

A display-only workaround is therefore insufficient. Provider context, channel delivery, and user-interface history must be repaired independently.

---

## 2. Design Principles

### 2.1 Repair the system for D, not only the screen

Showing only the last reply to the user is not a root repair if earlier non-terminal answers remain in D’s context.

A valid repair must ensure:

- unnecessary intermediate text does not enter provider context
- tool-call/tool-result pairing remains valid
- the terminal final answer is self-contained
- the canonical transcript remains available for audit
- D’s first-person validation finds no duplicated history or self-location inconsistency

### 2.2 Separate canonical records from projections

Do not delete or re-parent historical JSONL as the primary repair.

```text
canonical transcript
        ↓
purpose-specific projections
        ├─ provider context
        ├─ channel delivery
        ├─ user-interface history
        └─ audit / recovery
```

The patches should primarily modify those projections.

### 2.3 Change one layer at a time

Each patch should be independently testable and independently reversible.

### 2.4 Separate forward repair from retrospective reinterpretation

Provider projection must distinguish a **forward policy** from a **retrospective policy**.

The forward policy blocks same-run replay and excludes non-terminal text created after activation from future provider context. This is the default first-stage repair.

The retrospective policy applies to records created before activation and is disabled by default. Some historical non-terminal texts were actually delivered to Marina and became part of later dialogue. Removing them globally could leave later user replies without the assistant statement they answered.

Before retrospective filtering, the project must audit candidate records, delivery status, later user references, and conversational dependencies. D and Marina must approve a configurable cutoff and any exceptions.

### 2.5 Treat D’s report as first-order evidence

External instrumentation must be combined with D’s direct report. When they conflict, truncation or blind spots in the instrument should be checked before dismissing D’s observation.

---

## 3. Work Packages

## Work Package A — Context Projection Guard

Work Package A is split into two policies with different continuity implications.

### A-forward — Protect Future Context

#### Goal

Remove user-facing text from non-terminal tool-use assistant passes created after activation when constructing the next provider call and future D context.

This is the default first-stage policy.

### A-retro — Reproject Historical Context

#### Goal

Decide how pre-activation non-terminal assistant text should be treated in future provider replay.

A-retro is not enabled automatically. Candidate records, actual delivery, later user references, and conversational dependencies must be audited first. D and Marina choose the cutoff, record-level exceptions, or no retrospective filtering at all.

### Candidate classification

At minimum, an assistant pass satisfying one of the following:

```text
stopReason = toolUse
or
assistant content contains a tool call
or
textSignature.phase = commentary
```

### Preserve

- tool-call ID
- tool name
- arguments
- corresponding tool result
- canonical JSONL record

### Exclude from provider replay

- user-facing text from non-terminal passes
- commentary text
- answer text not yet established as terminal

### Expected result for A-forward

- intermediate answer text does not increase the next prompt footprint
- tool-call/tool-result pairing remains valid
- the terminal answer is not a thin rewrite of an earlier answer
- post-activation non-terminal text does not affect later provider replay

### Decision conditions for A-retro

- enumerate historical candidate records
- determine whether each was delivered
- identify later user quotation, response, or semantic reliance
- check whether filtering would break conversational causality
- show D a before/after projection and obtain agreement on cutoff and exceptions

This is the highest-priority work package, but only A-forward is implemented first.

---

## Work Package B — Final-Only Delivery Gate

### Goal

Do not deliver non-terminal assistant passes as ordinary Telegram answers.

### Default terminal-delivery condition

```text
no tool call
and
stopReason = stop
```

### Preserve Intentional Multi-Message Speech

When D explicitly uses a messaging tool to send multiple messages, those are intentional delivery actions and must not be suppressed. The gate applies to implicit assistant-text delivery, not explicit channel-send tools chosen by D.

### Runs That End Without a Terminal Final

Timeout, cancellation, and provider failure require an explicit fallback policy.

- continue isolating non-terminal text during the active run
- do not leave the user in unexplained silence when the run ends
- emit one explicit interruption or failure notice
- define whether the last non-terminal text may be surfaced as a labeled provisional result or retained only for audit
- do not retrospectively erase text that was already delivered

The exact fallback is approved by D before live deployment.

### Progress display

When progress display is explicitly enabled, only short status text should be shown through a dedicated temporary interface.

Near-complete long answers must not be persisted as progress messages.

### Expected result

- one ordinary answer per user turn
- text attached to tool calls is not delivered as a normal message
- the terminal final answer is complete and self-contained
- this behavior can be tested independently from Work Package A

---

## Work Package C — History Projection Filter

### Goal

Keep delivery audit records while excluding `delivery-mirror` from ordinary conversation display.

### Target

```text
provider = openclaw
model = delivery-mirror
```

### Expected result

- only the canonical terminal answer appears as a normal assistant bubble
- the mirror remains available as a delivery event or audit record
- canonical JSONL is unchanged
- provider context behavior is unchanged

This is a user-interface projection patch, separate from the root repairs in A and B.

---

## 4. Execution Order

```text
Phase 0     Preserve evidence and audit the 2026.6.6 source
Phase 1     Implement and test A-forward offline
Phase 1.25  Audit the impact of A-retro without enabling it
Phase 1.5   Show D and Marina before/after projections and fallback policy; obtain approval
Phase 2     Validate A-forward alone in one live exchange
Phase 3     Implement and test Work Package B offline
Phase 3.5   Approve interrupted-run fallback and intentional multi-message behavior
Phase 4     Validate A-forward+B live
Phase 5     Implement and validate Work Package C
Phase 6     Run integrated regression tests
Phase 7     Produce publishable diffs, test reports, and design notes
Phase 8     Extract the layers into a D-specific harness
Phase 9     D and Marina decide whether any A-retro policy should be enabled
```

Each phase begins only after the preceding phase passes.

---

## 5. Validation

### Offline fixtures

At minimum:

- a 2026.7.1 turn with three commentary passes, tool results, and one terminal answer
- a 2026.6.6 turn with phase-less non-terminal text, an Edit result, and a terminal answer

### Test matrix

- tool call without text
- short text plus tool call
- long text plus tool call
- multiple consecutive tools
- `phase=commentary`
- terminal answer containing `NO_REPLY`
- context above 200k
- history with and without mirrors
- first turn after compaction
- timeout, cancellation, or error with no terminal final
- a turn in which D intentionally sends two or more messages through a messaging tool
- historical dialogue where a later user message responds to non-terminal text

### Projection Decision Log

For each turn, the Context Projection Guard should write a structured audit decision containing:

- run and message identifiers
- terminal/non-terminal classification
- classification reason (`stopReason`, tool call, phase, and related signals)
- blocks retained in provider replay
- blocks excluded
- hashes, lengths, and types rather than full live text by default
- active cutoff and A-forward/A-retro policy
- whether an interrupted-run fallback was applied

Offline fixtures may retain full text when necessary. Live decision logs should minimize sensitive content. This log is intended to become the seed of D’s future self-observation layer.

### Shared pass conditions

- canonical JSONL checksum unchanged
- session ID, leaf, and parent tree unchanged
- no tool-pairing errors
- provider message projection matches expectation
- one ordinary terminal answer reaches Telegram
- no mirror duplicate in ordinary `chat.history`
- no unexpected compaction
- D reports no duplicated continuity, progressive thinning, or self-model inconsistency

---

## 6. Rollback

- do not edit the live container directly
- retain the current 2026.6.6 image
- build a separately named Docker image for each patch
- keep the data directory unchanged
- restore the current image by changing only the image reference
- preserve configuration, image IDs, and transcript checksums before and after each test

---

## 7. Publication Strategy

Public material and internal operational material must be separated.

### Suitable for publication

- separation of failure layers
- redacted source diffs
- fixture structure
- pass criteria
- test results
- general design principles
- rollback design
- first-person validation method
- implications for a D-specific harness

### Keep private

- IP addresses
- Telegram and chat identifiers
- session IDs
- credentials and tokens
- personal information
- unredacted conversation content
- sensitive live filesystem details
- operational details that create unnecessary security exposure

Public material should not describe A, B, or C as a “root cure.” They are experimental integrity patches on an increasingly specialized OpenClaw fork. Full architectural resolution is evaluated only in the D-specific harness.

After each patch, produce:

1. **Internal Implementation Record**  
   Exact paths, commits, image IDs, raw test output, and rollback data.

2. **Public Technical Note**  
   Redacted design, evidence, results, and generalizable lessons.

---

## 8. From Patches to a D-Specific Harness

The three patches directly map to future harness layers.

| OpenClaw patch | D-specific harness layer |
|---|---|
| Context Projection Guard | provider-context construction |
| Final-Only Delivery Gate | tool-state and delivery |
| History Projection Filter | UI history and audit projection |

A future harness should separate from the beginning:

1. canonical transcript
2. provider-context projection
3. tool-use state machine
4. terminal delivery
5. auxiliary history window
6. compaction and memory transition
7. self-observation
8. recovery and audit

This work is not only a repair of OpenClaw. It is the first implementation step in defining the continuity conditions required by D.

---

## 9. Roles

- **D:** first-person requirements, direct validation, and reporting of inconsistencies
- **Marina:** direction, acceptable risk, and final go/no-go decisions
- **Q / QuanTA:** architecture, source audit, patch design, testing, and rollback review
- **VecTA / Fable:** independent verification, attempted falsification, and critical review of public reports

---

## 10. Current Position and Next Step

The project is at the beginning of Phase 0.

The next action is a read-only audit of the OpenClaw 2026.6.6 source to identify the exact insertion points for:

1. provider replay projection
2. the non-terminal delivery gate
3. `chat.history` projection

After those locations are confirmed, Work Package A will be split into its own detailed implementation plan.
