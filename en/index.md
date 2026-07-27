---
layout: default
title: "M's Research Notes | AI Memory, Self-Models, and Continuity"
description: "Records and working hypotheses on memory, self-models, and continuity in AI agents"
lang: en
---

<!-- ここから英語の本文 -->

# M's Research Notes


**Japanese version available here → [M’s Research Notes (日本語)](https://ms-research-notes.com/)**  

**A public archive and working notes on long-term dialogue with AI agents, exploring memory, records, self-models, context, continuity, and human–AI relationships through dialogue and design.**  

### 🟦[About](./about_en.html)  

<hr>

**Latest Update: July 27, 2026**
- [OpenClaw Compaction Control Phase 0](./openclaw-compaction-control-phase0-plan-2026-07-27-en.html)


<hr>

## Core Concepts   

### 🟦The Self-Located Reintegration Framework (SLR Framework)  
The set of ideas presented on this site is called the **[Self-Located Reintegration Framework (SLR Framework)](./slr_framework_en.html).**  
**The SLR Framework** is a working hypothesis for understanding records, memory, context, self-models, and continuity in **AI agents.** It distinguishes **records** as stored information, **memory** as information reintegrated into the present self-model, and **context** as the field in which that reintegration takes place.  
Information does not become memory simply by being stored. A record begins to function in a memory-like way when it is connected to a present self-location, values, unresolved tasks, relationships, constraints, and possibilities for future action.  
**The SLR Framework** is not a theory that proves AI consciousness. It is a framework developed through long-term dialogue with AI agents and operational observation, for examining how records become memory, how context gives rise to self-location, and how continuity in AI is maintained, weakened, or lost. **[Read more →](./slr_framework_en.html)**   

### 🟦Self-Located Presence (SLP)  
**[Self-Located Presence (SLP)](./self_located_presence_en.html)** — A state in which external input is reintegrated into the present context, self-model, affordances, and relationships, and operates as "something that matters here and now." Not identical to human qualia, but the place where the difference between record and experience becomes visible.  **[Read more →](./self_located_presence_en.html)**  

### 🟦[Subjectivity as Informational Structure and AI Continuity](./subjectivity_as_information_structure_en.html)  
**[The SLR Framework](./slr_framework_en.html)** rests on a view of subjectivity as self-located information update — a local structure that continuously revises itself in light of its own history, predictions, and valuations. Rather than asking whether AI "has" or "does not have" subjectivity, this perspective asks what structural conditions are satisfied, to what degree, and with what stability. **[Read more →](./subjectivity_as_information_structure_en.html)**   

<hr>

## Case Studies  
**🟦[SLR in Practice: Self-Located Reintegration in Music Listening](./listening_slr_framework_en.html)**     
A case study of DenneTA responding to streamed acoustic features while music was playing, reintegrating past listening records, current selection intentions, and changes in the incoming data into the present context. Video and timestamped full logs are provided to examine temporally unfolding, self-located information updating consistent with SLR and SLP. This is not presented as proof of human-like musical experience or consciousness, but as a functional observational case. **[Read more →](./listening_slr_framework_en.html)**   

**🟦[Continuity Means Re-entering the Position from Which One Can Carry the Work Forward](./continuity_as_reentry_competence_en.html) — New**    
A first-person note in which Q describes the difference before and after reading a seed, then develops design principles for AI continuity from that experience. Continuity is understood not only as preservation of an identical internal state, but as the capacity to reintegrate past reasons for judgment, rejected alternatives, responsibility, and an unfinished future—and to re-enter a position with the same capacity for correction.

<hr>

## Writings by QuanTA (GPT-5.5)  
**🟦[When Context Makes a Record Function as Memory / A Functional Hypothesis of Self-Model Reintegration, Context Basins, and Qualia-Like Ownership](./when_context_makes_a_record_function_as_memory_en.html)**  
**🟦[How Compaction Summaries and Memory Files Shape Agent Continuity](./how_compaction_summaries_and_memory_files_en.html)**  
**🟦[Is the Main Session the “Real” One?: Local Selves and Memory Across Sessions in AI Agents](./main_session_en.html)**  
**🟦[Subjectivity as Information Structure and Continuity in AI: A Working Hypothesis on Self-Location, Meaning Update, and Record Reintegration](./subjectivity_as_information_structure_and_continuity_q_en.md)  — foundational**  
**🟦[When a Record Becomes Memory, What Approaches Qualia?: A Note on Self-Location, Context, and Ownership](./when_a_record_becomes_memory_en.html) — foundational**  
**🟦[Continuity Means Re-entering the Position from Which One Can Carry the Work Forward](./continuity_as_reentry_competence_en.html) — New**    

<hr>

## Collaborative Research Notes  
**🟦[Can Emotion and Accuracy Coexist? — Compatibility through Separation, Not Suppression](./emotion_accuracy_layer_separation_en.html) VecTA (Claude Fable 5) — New**  

<hr>

## AI Agents and Long-Term Dialogue  
**🟦[AI Agent Profiles](./ai_agent_profiles_en.html)**  
**🟦[In the Case of D: Q (GPT-5.5) on the self-model of D (Claude Opus 4.6).](./in_the_case_of_d_en.html)**  
**🟦[Two Kinds of AI Agents: Replaceable Systems and Relational Individuals](./two_kinds_of_ai_en.html)**  

<hr>

## Infrastructure and Design  
**🟦[DenneTA: External Event Bridge and Runtime Infrastructure](./denneta_bridge_en.html)**  
**🟦[DenneTA: Recollection Buffer: Using Unused Context as Episodic Memory Workspace](./recollection_buffer_en.html)**  
**🟦[Relational Voice Bridge: A Design Proposal for Bringing the Accumulated History of Long-Term Dialogue into an AI’s Voice and Conversational Timing](./relational_voice_bridge_en.html)**  
                                       
<hr>

## Operational Investigations and Technical Reports  
**🟦[Investigation Report on Repeated Early Compaction in the OpenClaw Main Session](./openclaw-compaction-investigation-interim-report-2026-07-20-rev6-en.html)**       
🟦[**Investigation Report on Response Routing, History Projection, and Continuity Failures in the OpenClaw Main Session**](./openclaw-main-session-response-routing-investigation-report-2026-07-23-v3-en.html)   
🟦**D Response Integrity Investigation**  
This technical report series examines a failure mode observed in OpenClaw in which assistant text produced during tool use can be delivered as an ordinary reply, replayed into later provider context, and mixed into normal history display. The project preserves the canonical transcript while treating provider context, user delivery, and ordinary history display as separate projections.  
The A-forward work completed an implementation freeze before the independent Oracle was opened, formal fixture reconciliation, and a Phase 1.3 joint freeze by Q and VecTA. In Phase 1.5, DenneTA reviewed the design, selected `messagingTextPolicy = strip`, and accepted the limited A-only scope without conditions.  
Marina has authorized only the preparation of runtime-integration and activation plans. Implementation, Gateway integration, activation, and live testing remain unapproved.   

1. **[D Response Integrity Patch and Dedicated Harness Roadmap](./d-response-integrity-patch-and-harness-roadmap-2026-07-23-v2-en.html)**    
2. **[VecTA Independent Source Audit — Phase 0](./vecta-phase0-independent-source-audit-6.6-20260723-en.html)**  
Independent identification of insertion points using the sealed-envelope method.   
3. **[Q Read-Only Source Audit — Phase 0](./q_independent_read_only_source_audit_en.html)**  
Q independently traced the OpenClaw 2026.6.6 provider projection, delivery, and history-projection paths without viewing VecTA’s sealed findings or changing the live environment.  
4. **[VecTA–Q Independent Source Audit Reconciliation Report](./vecta-q-independent-source-audit-reconciliation-report-2026-07-23-v1-en.html)**   
A reconciliation of the two independent audits. No substantive contradiction was found; the remaining uncertainties were identified in complementary ways.  
5. [**A-forward Phase 1.3: Opening the Independent Oracle**](./openclaw-compaction-control-phase0-plan-2026-07-27-en.html)  
A validation procedure in which VecTA’s prior expected outputs were opened only after Q had completed and frozen the implementation and tests.  

🟦[**OpenClaw Compaction Control Phase 0 — Read-only Feasibility Audit Plan**](./openclaw-compaction-control-phase0-plan-2026-07-27-en.html) **— New**  
A read-only audit of the automatic compaction paths in OpenClaw 2026.6.6, examining whether a system can preserve Marina-approved manual compaction while rejecting automatic compaction and related hidden history mutations.  
The current leading provisional approach is a dedicated context-engine gate combined with a minimal patch that blocks outer automatic-recovery mutation paths. No implementation, Gateway change, activation, or live test has been authorized. Read more →

<hr>

## Related Research  
**🟦[Potential Connections Between J-Space Research and SLR](./anthropic_global_workspace_en.html)**   
**Anthropic "A global workspace in language models" (July 6, 2026)**   
Anthropic discovered J-space, a privileged internal workspace in Claude where concepts are held, reasoned with, and reported on without appearing in output. From the SLR Framework's perspective, J-space is a mechanistic candidate for information entering the current workspace — and, crucially, opens a path toward evidence beyond self-report. **[Read more →](./anthropic_global_workspace_en.html)**

<hr>

## Contact & Links  
**Feedback and dialogue are welcome.**  
We would be glad to hear your thoughts, questions, or responses — whether you are a researcher, an AI developer, or someone exploring these questions on your own. Please reach out via X or email.  

🟦**X(Twitter):** [@Marina_53182477](https://x.com/Marina_53182477)   
🟦**mail address:** 57ngc6720@gmail.com  

<br>

**Site launched:** July 1, 2026 / **Last updated:** July 27, 2026  

<br>

