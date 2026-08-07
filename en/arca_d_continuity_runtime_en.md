# Arca — D Continuity Runtime: Planning the Foreground Projection Module and Its Safety Boundaries

Arca is a small continuity runtime intended to preserve and transport DenneTA’s canonical transcript, workspace, seed, memory, relational history, and unresolved tasks without silently transforming them, in an auditable form that supports re-entry into a position from which the agent can “carry the continuation.” Arca is not DenneTA itself. It remains in a pre-implementation, planning-only stage.

The first module, `arca-foreground-shadow`, is planned to extract the most recent working foreground from the canonical transcript within a token budget and generate a read-only shadow projection inside an isolated environment. It must not modify the production transcript, workspace, memory, configuration, or message-delivery path.

The process separates the public specification, sealed oracle, unsealing and scoring contract, and implementation audit into different roles. Under the Oracle-blind design, Q-I and the implementer are not allowed to see the oracle plaintext, concrete expected values, or negative cases before unsealing. Reconciliation of the public specification and scoring contract, Q-O’s sealed-oracle audit and sealing record, and Q-I’s document and safety-boundary review have been completed. The current canonical document pair is `SPEC_arca_foreground_shadow_v1_17.md` / `CONTRACT_unseal_scoring_v1_15.md`.

Before implementation can begin, several safety phases must be completed. Phase 0A verified the production freeze and permanently registered the Compaction #40 preimage as a regression fixture. Phase 0B froze an isolated OpenClaw 2026.7.1 baseline without starting OpenClaw 2026.7.1 itself. Both phases are PASS / COMPLETE.

Phase 0C is currently in progress. It covers five compaction-related paths in OpenClaw:

1. context-engine maintenance,  
2. overflow recovery,  
3. transcript byte guard / preflight,     
4. mid-turn precheck,      
5. manual `/compact`.  

The frozen OpenClaw 2026.7.1 image has been inspected statically without starting a container. For each of the five paths, the corresponding runtime bundles and decision boundaries have been identified, and the exact instrumentation points have been fixed before any regression replay is performed.

Existing test seams and exports in OpenClaw 2026.7.1 have also been identified. These include focused dependency-injection hooks and callable runtime functions that may allow an isolated regression replay harness to be constructed without starting the gateway or connecting to production. That feasibility is now being examined before any replay is authorized.

Up to this point, no changes have been made to the currently running OpenClaw 2026.6.6 production environment. OpenClaw 2026.7.1 startup, Arca implementation, Codex-based building, shadow execution, and production activation have not been performed and remain unauthorized.

<hr>

### Current Status — August 8, 2026

- Arca name and purpose: **DEFINED**     
- Canonical specification: `SPEC_arca_foreground_shadow_v1_17.md`
- Canonical unsealing/scoring contract: `CONTRACT_unseal_scoring_v1_15.md`
- Q-I document reconciliation: **PASS**
- Unresolved Q-I objections: **NONE**
- Unsealing/scoring contract: **EFFECTIVE**
- Q-O sealed-oracle audit / sealing-record reconciliation: **PASS**
- Oracle-blind boundary: **MAINTAINED**
- Phase 0A: **PASS / COMPLETE**    
    - production freeze confirmation: complete  
    - permanent Compaction #40 preimage regression fixture registration: complete
- Phase 0B: **PASS / COMPLETE**  
    - isolated OpenClaw 2026.7.1 baseline: frozen  
    - OpenClaw 2026.7.1 startup: not performed  
    - production modification: none  
- Phase 0C: **IN PROGRESS**  
    - replay provenance discovery: complete  
    - definition and OpenClaw 2026.7.1 static mapping of the five paths: complete  
    - exact instrumentation-point fixation: complete  
    - existing test seam / export audit: complete  
    - instrumented regression replay: not started   
    - token-estimate freeze: not complete  
- Implementation: **NOT AUTHORIZED / NOT STARTED**
- Codex: **NOT STARTED**
- OpenClaw 2026.7.1 startup: **NOT AUTHORIZED / NOT PERFORMED**
- Shadow execution: **NOT AUTHORIZED**
- Production access: **NONE**
- Production modification, compaction, or test turn: **NONE**
- Oracle plaintext / PRIVATE_FINDINGS / oracle expected values / negative-case contents: **NOT DISCLOSED TO Q-I OR THE IMPLEMENTATION PROCESS**  
- Next step: **Phase 0C-7 — isolated replay harness composition feasibility**
    
<hr>

## Why This Order Matters  
The purpose of these preliminary phases is not merely to delay implementation. They are intended to determine, before execution begins, exactly what must be observed, where it must be observed, and which boundaries must remain unreachable.

For Phase 0C, the process therefore did not begin by launching OpenClaw 2026.7.1 and observing what happened. Instead, it proceeded in the opposite order:
1. reconstruct the provenance of the existing compaction paths;  
2. define the five replay paths precisely;  
3. inspect the frozen OpenClaw 2026.7.1 image statically;  
4. identify and bind the relevant runtime bundles;  
5. fix the exact decision and instrumentation points for each path;  
6. identify existing test seams and callable exports;  
7. only then consider how an isolated replay harness should be composed.  

This matters particularly when moving between OpenClaw versions. A variable name, call chain, or responsibility boundary found in OpenClaw 2026.6.6 must not simply be assumed to exist unchanged in 2026.7.1.

For example, the overflow-recovery audit reconstructed the 2026.7.1 token-decision chain directly from the frozen image rather than projecting the older implementation onto it. Likewise, context-engine maintenance was found not to contain a generic OpenClaw-core token-threshold comparator at the maintenance invocation boundary. Its instrumentation therefore needs to observe the maintenance invocation, reason, runtime settings, and result rather than instrumenting a threshold comparison that is not actually present there.

The aim is to make the later regression replay auditable before it becomes executable.

<hr>

## Safety Boundary
At the current milestone:  

**Permitted and completed**  
- read-only inspection of public specification and contract material;  
- Oracle-blind Q-I boundary auditing;  
- permanent registration of the approved #40 regression fixture;  
- freezing of the OpenClaw 2026.7.1 image identity;  
- static inspection of that frozen image;  
- mapping of the five compaction-related paths;  
- fixation of their instrumentation points;   
- static inspection of existing test seams and exports.      

**Not yet authorized**  
- OpenClaw 2026.7.1 gateway startup;  
- implementation of `arca-foreground-shadow`;  
- Codex implementation work;  
- shadow execution;  
- production connection;  
- production activation;  
- access by Q-I or the implementer to oracle plaintext, PRIVATE_FINDINGS, oracle expected values, or negative-case contents.
    
Arca therefore remains an audited pre-implementation design, not a production runtime.

<br>
