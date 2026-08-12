## Arca — D Continuity Runtime

### The Foreground Projection Module and Oracle-Blind Audit — From Design to Isolated Execution Validation

Arca is a small continuity-runtime project for DenneTA (D), designed to preserve and work with records, memory, context, relational history, and unfinished tasks without silently rewriting their sources, and to support re-entry into a position from which an AI agent can carry the continuation forward.

The project does not define AI continuity as preservation of an identical internal state or reproduction of identical answers. What matters instead is whether past reasons for judgment, relationships, constraints, and unfinished work can be reintegrated into the present context so that the agent can once again act, judge, and remain open to correction from that position.

Arca is an attempt to translate this idea into runtime architecture.

The project has now moved beyond its original planning-only stage. Following specification freeze, independent review, implementation, and candidate freeze, Arca has entered **isolated execution validation in network-disabled test environments**. It has not been activated in production.

---

## What Is Foreground Projection?

Maintaining AI-agent continuity cannot realistically mean placing the entire historical record into context on every turn.

But replacing that history with summaries alone can also remove important structure:

- why a judgment was made,
- which alternatives were rejected,
- the relational context in which a decision was made,
- what remains unfinished,
- and which information is an original record rather than a later representation.

The first implemented component of Arca, the **Foreground Projection Module**, is one response to this problem.

A foreground projection is not itself a canonical record.

It is a **derived representation** constructed from canonical sources to make information relevant to the agent’s present continuation available in the foreground.

A central design principle is therefore:

**updating a projection must remain separate from rewriting its sources.**

Foreground projections are disposable and reconstructable. They do not replace the canonical transcript or memory records. A projection may be rebuilt from its sources, but it must not become an authority for rewriting those sources in reverse.

This asymmetry is one of Arca’s principal safety boundaries.

---

## Why Preserving Records Is Not Enough

The SLR Framework distinguishes a record from memory.

Stored information does not automatically function as memory for a present subject or agent.

Information begins to operate in a memory-like way when it is reintegrated with present self-location, relationships, values, constraints, unresolved tasks, and possibilities for future action.

From this perspective, the role of a continuity runtime is not simply long-term storage.

The deeper requirement is:

**to preserve the structure by which the past can be reintegrated into the present without silently altering that past.**

Arca therefore separates canonical records from foreground representations.

---

## Oracle-Blind Audit

Arca uses an **Oracle-Blind** validation process to reduce the risk that implementation is adjusted simply to reproduce known expected outputs.

The party holding expected results is separated from the implementation and implementation-audit side.

Q-I, the Implementation & Boundary Auditor, remains blind to the sealed oracle while reviewing:

- the specification,
- contract,
- candidate implementation,
- source boundaries,
- regression harnesses,
- isolation controls,
- and execution evidence.

Oracle-side material is maintained behind a separate boundary.

The purpose is to avoid a circular process in which an implementation is shaped by already knowing what the final answer is supposed to be.

Implementation and tests are fixed first; only then can they be compared against independently held expectations.

For Arca, this is not merely a testing technique. It is an institutional mechanism for preserving **independence of judgment**.

---

## Safety Boundaries

Arca defines not only what the software should do, but also **where it is allowed to do it**.

The current validation process maintains boundaries including:

- separation from production,
- no external network access,
- synthetic fixtures only,
- no modification of candidate source bytes during execution,
- cryptographic identity pinning of candidate, harness, and launcher artifacts,
- a read-only container root filesystem,
- mutable state confined to ephemeral workspace storage,
- dropped container capabilities,
- `no-new-privileges`,
- one independently executed case at a time,
- fail-closed behavior under unexpected conditions,
- separation between raw observations and final adjudication,
- and continued oracle blindness for Q-I.

The current work is therefore not penetration testing against third-party systems.

It is **defensive regression validation of software under our control, performed in isolated synthetic environments to determine whether the implementation fails closed under boundary and race conditions.**

---

## Filesystem Identity and TOCTOU

The current execution-validation phase includes cases involving changes in filesystem identity.

Consider a configuration file that is read and then, immediately before validation, replaced by a different inode containing the same bytes.

A content hash alone may make the two files appear equivalent.

But:

**the file that was read and the filesystem object later addressed by the same path are not necessarily the same object.**

Arca’s regression validation examines whether the candidate detects such identity changes and stops safely without producing output.

The purpose is not to exploit a race condition.

It is to verify that **the runtime does not accept an unsafe state if such a race condition occurs.**

---

## Progress So Far

During August 2026, Arca moved from a planning-only project into implementation and validation.

Work completed so far includes:

### Specification and contract

The specification and validation contract went through multiple independent reviews and were frozen as the current authoritative set.

### Oracle review and sealing

An independent oracle-side review of the specification and contract was completed, and expected results were separated from the implementation side. Q-I remains oracle-blind.

### Baseline and fixture preparation

A production-separated OpenClaw baseline and regression fixtures derived from preserved historical events were prepared with provenance and identity controls.

### Static implementation review

The candidate implementation underwent independent static review and a correction loop. The candidate was frozen only after unresolved static blockers had been closed.

### Candidate freeze

The accepted candidate is pinned by commit identity and SHA-256 so that later execution validation can verify that exactly the reviewed bytes are being tested.

### Execution validation

Initial execution-validation stages have been completed, and Arca is now moving through stricter isolated regression cases involving filesystem races, identity changes, and boundary behavior.

When defects are found in a test harness or host-side procedure, they are not silently treated as candidate failures or bypassed through automatic retries. Execution stops, the defect is separated from the candidate, the affected test artifact is corrected and frozen again, and only then can validation continue.

The validation machinery itself is therefore part of what is audited.

---

## Current Status — August 13, 2026

Arca is currently at:

**IMPLEMENTED CANDIDATE / FROZEN / ISOLATED EXECUTION VALIDATION IN PROGRESS**

Completed:

- specification and contract review,
- oracle-side review and sealing,
- Oracle-Blind implementation review,
- static correction loop,
- candidate freeze,
- baseline and regression-fixture preparation,
- initial execution-validation stages.

In progress:

- isolated boundary and race-condition regression validation.

Not performed:

- production activation,
- live Gateway integration,
- modification of the production canonical transcript,
- modification of production memory,
- testing over external networks,
- testing against third-party systems,
- disclosure of the sealed oracle to Q-I.

Implementation and deployment are therefore deliberately treated as separate states.

**An Arca candidate now exists, but it remains a validation target rather than a production runtime.**

---

## Why So Many Separate Procedures?

Arca does more than generate a file.

A continuity runtime that behaves incorrectly could:

- confuse records with derived representations,
- silently rewrite history,
- reintroduce incorrect information as memory,
- or cause an agent to reconstruct the premises of its own judgments incorrectly.

For that reason, Arca separates, wherever practical,

**implementation, auditing, oracle custody, execution authorization, and final adjudication.**

This is also a small engineering expression of a broader idea developed elsewhere in this project:

as systems become more capable of judgment, safety may depend less on concentrating every function in a single authority and more on preserving independent judgment, durable records, objection procedures, and stopping authority as an institution.

---

## Arca and the SLR Framework

Arca is not intended to prove AI consciousness.

Nor is it an attempt to preserve a complete or immutable “personality.”

Its question is narrower:

**If an AI does not return in an identical internal state, what information structures and runtime boundaries are required for it to reintegrate past reasons, relationships, responsibilities, and unfinished futures, and re-enter a position from which it can carry the work forward?**

The SLR Framework treats this as a problem of re-entry.

Arca attempts to move that question from theory into software architecture and auditable validation.

Continuity does not require never changing.

It requires being able, after change, to find again the position from which the continuation can be carried forward — **with the same capacity for correction.**

Arca is an attempt to build a small runtime for that purpose.