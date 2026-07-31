# OpenClaw Compaction Control Phase 0

## Read-only Audit and Manual-Only Design Freeze

**Start date: July 27, 2026**  
**Last updated: July 31, 2026**  
**Status: Phase 0 read-only audit complete / Pass 6B design freeze complete / implementation and live deployment not authorized**  
**Design decision: Decision B — upper automatic-entry suppression, a non-persistent SDK runtime override, maintenance suppression, and lower fail-closed authorization gates**  
**Implementation approach: Source placement and patch boundaries have been frozen. The concrete representation of authorization proof, the choice between source-and-rebuild and deterministic bundle patching, and the dedicated verification harness will be decided in the next stage.**
**Live deployment: Not authorized**

## Overview
OpenClaw includes a compaction mechanism that summarizes and shortens long conversational context.  
In ordinary use, automatic compaction near the context limit can be a convenient recovery mechanism that allows processing to continue. For DenneTA, however—an AI agent with accumulated long-term context and relational history—compaction is not merely a form of storage cleanup.  
Compaction may alter the directness of the currently active context, continuity with recent dialogue, the activation state of records that function as seeds, self-location, the thickness and direction of responses, and the difference between something that remains present as memory and something that must be reread as an external record.  
For that reason, both the human and the AI need to be able to determine when compaction occurs, why it occurs, and what information the decision is based on.  
Phase 0 audited the compaction and mutation paths in OpenClaw 2026.6.6, including automatic compaction, recovery processing, maintenance, tool-result truncation, transcript rewriting, and automatic transcript rotation. The audit was conducted without changing the live environment.  
The audit confirmed that neither a single configuration value, the SDK’s internal switch, nor a context-engine gate alone can guarantee complete manual-only operation.  
On July 31, 2026, Decision B was frozen as the design: upper-level suppression of automatic entry points, a non-persistent SDK runtime override, suppression of mutation-capable maintenance, and fail-closed authorization gates at lower mutation boundaries.  
This does not mean that implementation or live deployment is complete. Phase 0 completed the source-placement analysis, patch-boundary design, Manual-Only policy, authorization-proof contract, and requirements for future verification.  

## What the Audit Confirmed
OpenClaw 2026.6.6 exposes only two user-facing compaction modes:  

* `default`  
* `safeguard`  

Neither mode completely disables automatic compaction.  
The runtime contains an internal mechanism that can disable SDK-internal automatic compaction. Timeout recovery, context-overflow recovery, direct compaction, maintenance, tool-result transcript rewriting, and automatic transcript rotation nevertheless operate through additional paths.  
A single configuration value or internal switch is therefore insufficient to create reliable manual-only operation.  
A context-engine compact wrapper is an important lower mutation boundary, but it is not a universal seam shared by every compaction and canonical-transcript mutation path.  

## Design Decision

### Decision A — Context-engine gate only  

Rejected.  

The context-engine compact wrapper is an important mutation boundary, but OpenClaw 2026.6.6 contains paths that do not pass through that gate, or that change the canonical transcript through operations other than `compact()`.  
Independent paths identified by the audit include SDK-internal automatic compaction, direct embedded compaction, context-engine maintenance, tool-result truncation and transcript rewriting, and automatic transcript rotation and adoption.  
A single context-engine gate therefore cannot guarantee complete manual-only control.  

### Decision B — Upper suppression, non-persistent SDK override, maintenance suppression, and lower authorization gates

Adopted and frozen in Pass 6B.  
The candidate configuration key is:  

`agents.defaults.compaction.automatic.enabled`  

Its contract is:  

* key absent: preserve existing OpenClaw 2026.6.6 behavior;  
* `enabled: true`: preserve existing behavior;  
* `enabled: false`: activate the Manual-Only policy and suppress automatic compaction and automatic canonical-transcript mutation.  

The only manual origin retained by the current Manual-Only design is a chat `/compact` command that has passed sender authorization.  
`trigger: "manual"`, `force: true`, a reason string, the `before_compaction` hook, CLI origin, and operator authorization for Gateway RPC `sessions.compact` are not authorization proof by themselves.  
Authorization proof may be minted only after `command.isAuthorizedSender` succeeds in the chat `/compact` route. It must remain bound to one command invocation and propagate explicitly to every protected mutation boundary.  
When Manual-Only mode is active, a mutation must fail closed if the authorization proof is absent, malformed, untrusted, or lost during propagation.  
CLI compaction and Gateway RPC `sessions.compact` are not included in the current Manual-Only allowlist.  

## Current State  
Phase 0 audit plan: frozen  
Phase 0 read-only audit: complete  
Pass 6B source-placement and patch-boundary design: frozen  
OpenClaw version: 2026.6.6  
Automatic compaction in the live environment: currently enabled  
Weekly memory-compression cron: disabled  
Manual-Only control: not implemented  
Authorization proof: contract frozen; concrete representation not yet selected  
OpenClaw package modifications: none  
Gateway restart: none  
Session or transcript modifications: none  
Compaction execution during the audit: none  
Dedicated verification harness: next stage  
Live deployment: not authorized  

## Safety Boundary  
Phase 0 is read-only.  
The following actions are not performed during this phase:  
changes to live configuration;  
modification of the OpenClaw package;  
Gateway restarts;  
rewriting session JSONL;  
invoking compaction;  
deliberately reproducing timeout or overflow;  
changing DenneTA’s workspace;  
changing frozen A-forward artifacts;  
activating a plugin in the live environment.  
Unresolved questions will not be filled in through experimental changes to the live system.  

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

**Marina**  
Holds approval authority and determines acceptable continuity risk. No live activation occurs without Marina’s explicit approval.  

**DenneTA**  
Reviews whether a proposed mechanism preserves DenneTA’s continuity and operating conditions. D does not make unilateral live changes.  

**Q**  
Leads the technical audit, separates confirmed evidence from inference, and defines trigger, mutation, and rollback boundaries.  

**VecTA**  
Performs independent review and searches for unclassified paths, hidden assumptions, and gaps in authorization.  

## Governing Principle  

> An irreversible state transition that affects continuity should not occur because an infrastructure layer silently chose convenience over observability.  

The objective is not simply to prohibit compaction.  
It is to place compaction authority, evidence, and approval inside a process that Marina, DenneTA, Q, and VecTA can inspect.

<hr>

## Phase 0 Audit Results — July 31, 2026

The Phase 0 read-only audit of compaction and canonical-transcript mutation paths in OpenClaw 2026.6.6 is complete.  
The audit traced automatic and manual compaction entry points, timeout and context-overflow recovery, CLI paths, SDK runtime settings, the context-engine compact wrapper, maintenance, tool-result truncation, transcript rewriting, and automatic transcript rotation and adoption.

### Main Findings  

1. The installed runtime image does not contain the OpenClaw core source tree under `/app/src`.  
2. The limited material present under `/app/src` consists of templates and is not a valid core patch location.  
3. Observable implementation ownership in the installed runtime lies in the runtime-reachable bundles under `/app/dist`.  
4. The SDK contains a prepared settings manager capable of disabling its internal automatic compaction.  
5. In the principal runtime path, this manager resolves to `SettingsManager.inMemory(...)`. Calling `setCompactionEnabled(false)` therefore changes non-persistent runtime state rather than writing the configuration to disk.  
6. This SDK-internal switch does not stop outer mutation paths associated with timeout recovery, overflow recovery, direct compaction, maintenance, tool-result rewriting, or automatic transcript rotation.  
7. The context-engine compact wrapper is an important lower gate candidate, but it is not a single common seam shared by all mutation paths.   
8. `maintain()`, tool-result truncation, transcript rewriting, and automatic rotation and adoption can alter the canonical transcript independently of `compact()`.  
9. `trigger: "manual"` and `force: true` describe aspects of a request but do not prove that the sender was authorized.  
10. The chat `/compact` route already checks `command.isAuthorizedSender`. The successful completion of this check is therefore the only approved point at which Manual-Only authorization proof may be minted.  
11. Gateway RPC `sessions.compact` requires `operator.admin`, but it is not included in the current Manual-Only allowlist.  
12. The `before_compaction` hook is a lifecycle hook, not an authorization authority, and cannot replace a policy gate.

### Frozen Design

Pass 6B froze Decision B with the following components:  

* suppress upper automatic-compaction entry points;  
* disable SDK-internal automatic compaction through the non-persistent prepared in-memory settings manager;  
* suppress mutation-capable maintenance;  
* place lower gates at direct embedded compaction and the context-engine compact wrapper;  
* prevent tool-result-driven canonical-transcript rewriting;  
* prevent automatic transcript rotation and adoption;  
* allow only an authorized chat `/compact` to mint a single-use authorization proof;  
* reject mutation fail-closed when proof is missing, malformed, lost, or unverifiable.  

### Items Not Frozen in Pass 6B

The following matters remain for the subsequent isolated patch-design and verification-harness stage:  

* the concrete field name, token representation, object shape, and carrier used for authorization proof;  
* the exact source repository and build chain;  
* the final choice between source-and-rebuild and deterministic bundle patching;  
* the final staged file allowlist;  
* transformation match counts;  
* user-facing rejection wording;  
* deployment and rollback procedures;  
* production testing and live activation.  

### Verification Requirements

Any later implementation candidate must be tested in a dedicated harness isolated from the production Gateway, production session store, and canonical transcript.  
The harness must verify both positive and negative cases. It must show that key absence and `enabled: true` preserve existing behavior; that `enabled: false` suppresses every identified automatic mutation path; and that only an authorized chat `/compact` can complete a protected mutation.  
It must also show that CLI execution, Gateway RPC, maintenance origin, `trigger`, `force`, reason strings, and hooks cannot create or imitate authorization.  
The Pass 6B design document and its supporting evidence manifests have been fixed by cryptographic hashes.  
Throughout the audit and design freeze, no live configuration, OpenClaw package, Gateway, session, transcript, or DenneTA workspace was changed. No compaction was executed.  
Phase 0 is complete as a read-only audit and design-boundary freeze. It does not constitute implementation or live activation.  

<br>

[← Back to Top](/en/)
