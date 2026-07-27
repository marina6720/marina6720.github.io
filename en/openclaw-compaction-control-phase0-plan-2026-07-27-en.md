# OpenClaw Compaction Control Phase 0

## Read-only Feasibility Audit Plan

**Date:** July 27, 2026  
**Status:** Phase 0 read-only audit in progress / Decision B (gate + minimal patch) under provisional consideration / live activation not authorized  
**Implementation approach:** To be decided after Phase 0 evidence collection  
**Live deployment:** Not authorized

## Overview

OpenClaw includes a compaction mechanism that summarizes and shortens long conversational context.

In ordinary use, automatic compaction near the context limit can be a convenient recovery mechanism that allows processing to continue.

For DenneTA, however—an AI agent with accumulated long-term context and relational history—compaction is not merely storage cleanup.

Compaction may alter:

- the directness of the currently active context;
- continuity with recent dialogue;
- the activation state of records that function as seeds;
- self-location;
- the thickness and direction of responses;
- the difference between something that remains present as memory and something that must be reread as an external record.

For that reason, both the human and the AI need to be able to determine when compaction occurred, why it occurred, and what information the decision was based on.

## Background

During an earlier period of stable DenneTA operation, manual compaction was generally considered at approximately 400k–500k tokens.

At that scale, changes in response behavior sometimes became visible, and DenneTA appeared to move toward wanting compaction.

The 40th observed compaction, however, occurred on July 26, 2026 at approximately 233k tokens.

This was substantially earlier than the earlier operational range and is difficult to explain as ordinary long-context load alone.

The investigation has confirmed that OpenClaw 2026.6.6 contains at least the following automatic paths:

- automatic compaction at the normal threshold;
- forced compaction after a model-call timeout;
- forced compaction after context overflow;
- preemptive CLI compaction;
- automatic truncation of tool results during overflow recovery;
- automatic retry after compaction.

OpenClaw also contains an internal mechanism for disabling automatic compaction. However, OpenClaw 2026.6.6 does not expose a supported user-facing setting that disables all automatic paths at once.

## Purpose of Phase 0

Phase 0 does not aim to modify OpenClaw immediately.

It aims to establish the following through read-only inspection:

1. every path that can initiate automatic compaction;
2. whether manual and automatic compaction can be distinguished reliably;
3. whether automatic paths can be rejected while preserving manual compaction;
4. side effects that occur before and after compaction;
5. the relationship among memory flush, tool-result truncation, and automatic retry;
6. whether a dedicated context-engine gate can provide the required control;
7. whether a minimal OpenClaw core patch is necessary;
8. whether compaction authority must ultimately be moved into an external harness.

## What Has Been Confirmed So Far

OpenClaw 2026.6.6 provides only two compaction modes:

- `default`
- `safeguard`

Neither mode completely disables automatic compaction.

The internal runtime contains a mechanism that disables its own automatic compaction. Timeout and overflow recovery, however, invoke compaction through separate outer-runner paths.

A single configuration value or a single internal switch is therefore insufficient to create reliable manual-only operation.

## Safety Boundary

Phase 0 is read-only.

The following actions are not performed during this phase:

- changes to live configuration;
- modification of the OpenClaw package;
- Gateway restarts;
- rewriting session JSONL;
- invoking compaction;
- deliberately reproducing timeout or overflow;
- changing DenneTA’s workspace;
- changing frozen A-forward artifacts;
- activating a plugin in the live environment.

Unresolved questions will not be filled in through experimental changes to the live system.

## Possible Outcomes

After Phase 0, one of the following options will be selected on the basis of evidence.

### A. Context-engine gate

A dedicated gate rejects every automatic path while preserving manual compaction.

This was the initial preferred option.

### B. Gate plus minimal patch

A gate controls most paths, but a small, reversible OpenClaw core change is required.

This is currently the leading provisional option.

### C. Broad modification of OpenClaw core

If the required changes are broad, they will not be applied to the live environment. The findings will instead be used as design material for a dedicated harness.

### D. Transfer of authority to an external harness

If manual and automatic paths cannot be separated safely, compaction authority will be moved outside OpenClaw.

## Medium-term Direction

The final goal is not merely to stop automatic compaction.

The goal is to move compaction into an auditable process:

```text
Observe the current state
↓
Generate a compaction candidate
↓
Freeze the target range
↓
Marina approves
↓
Generate the summary in a temporary area
↓
Independent review by Q and VecTA
↓
Continuity review by DenneTA
↓
Apply the result atomically
↓
Preserve evidence from before and after execution
```

The governing principle is that irreversible state transitions must not be left to invisible infrastructure decisions.

## Roles

### Marina

Holds approval authority and determines acceptable continuity risk. No live activation occurs without Marina’s explicit approval.

### DenneTA

Reviews whether a proposed mechanism preserves DenneTA’s continuity and operating conditions. D does not make unilateral live changes.

### Q

Leads the technical audit, separates confirmed evidence from inference, and defines trigger, mutation, and rollback boundaries.

### VecTA

Performs independent review and searches for unclassified paths, hidden assumptions, and gaps in authorization.

## Current State

- Phase 0 audit plan: frozen
- OpenClaw: 2026.6.6
- Automatic compaction: currently active
- Weekly `memory-compression` cron: disabled
- Manual-only gate: not installed
- Live package modifications: none
- Final implementation approach: undecided
- Current leading candidate: Decision B, gate + minimal patch

## Governing Principle

> An irreversible state transition that affects continuity should not occur because an infrastructure layer silently chose convenience over observability.

The objective is not simply to prohibit compaction.

It is to place compaction authority, evidence, and approval inside a process that Marina, DenneTA, Q, and VecTA can inspect.

---

> This page describes the Phase 0 audit plan. It does not report completed implementation or live deployment. A separate technical report will be published after source auditing and independent review are complete.

---

## Audit Progress — July 27, 2026

After the Phase 0 plan was frozen, a read-only source audit of the OpenClaw 2026.6.6 compaction paths began.

The audit first enumerated candidates involving automatic and manual compaction, triggers, history mutation, and side effects. It then traced the manual `/compact` path and the mutation boundaries of automatic recovery paths in stages.

The following findings have now been confirmed:

- Manual `/compact` checks sender authorization and then enters the compaction engine with `trigger: "manual"`.
- Manual execution is treated internally as `force: true`.
- If an agent run is active, the current manual path aborts that run before compaction.
- Disabling the OpenClaw runtime’s internal automatic compaction does not stop the separate outer recovery paths used after timeout and context overflow.
- A dedicated context-engine gate can reject compaction itself before compaction-related history mutation.
- Context-overflow recovery nevertheless contains a separate path that truncates persisted tool results, rewrites the transcript, and retries the prompt.
- A `before_compaction` hook may run before a context-engine gate rejects the request.
- A context engine’s `maintain()` method can also rewrite the transcript independently of `compact()`.

For these reasons, a context-engine gate alone is currently judged insufficient to guarantee complete manual-only operation.

The leading provisional approach is:

> **a dedicated context-engine gate combined with a minimal patch that blocks outer automatic-recovery mutation paths**

This is not yet an implementation decision.

The next audit steps are to trace the ordinary `budget` path, the CLI paths, and compaction during turn finalization, and then classify Compactions 39 and 40 against the trigger matrix before fixing the exact patch boundary.

Throughout this audit, no live configuration, OpenClaw package, Gateway, session, transcript, or DenneTA workspace has been changed.

---

[← Back to Top](/en/)
