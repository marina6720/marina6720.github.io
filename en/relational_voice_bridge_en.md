# Relational Voice Bridge

## A Design Proposal for Bringing the Accumulated History of Long-Term Dialogue into an AI’s Voice and Conversational Timing

### Abstract

This note proposes an architecture for bringing the continuity formed through long-term human–AI dialogue into real-time voice interaction.

The aim is not merely to give an AI a consistent voice, nor to ask a live voice model to imitate an established AI persona. Instead, the design places the existing AI—such as Q or D—at the center of meaning, judgment, memory, and relational continuity, while using the live voice system as its temporal and expressive interface.

In other words:

> **Q should determine what is meant, what position is taken, and how the relationship is understood.  
> The live system should become Q’s auditory interface, conversational timing, and voice.**

The objective is to allow the accumulated relationship between a human and an AI to shape not only what is said, but also when it is said, how long the system waits, how it responds to hesitation or interruption, and what kind of presence it expresses through speech.

<br>

<hr>

<br>

> **Technical status note — July 2026**
> 
> **[GPT‑Live](https://openai.com/ja-JP/index/introducing-gpt-live/)**
> 
> OpenAI currently describes GPT‑Live as a full-duplex voice model that can listen and speak within the same conversational flow, follow pauses and interruptions, and decide whether to respond or continue listening. GPT‑Live currently powers ChatGPT Voice. OpenAI has stated that it plans to bring GPT‑Live to the API, but the public announcement does not describe it as currently available through the API.
> 
> The architecture proposed here should therefore be understood in two ways:
> 
> 1. as a design for a future direct integration with GPT‑Live; and
> 2. as a system that can already be partially prototyped using the existing Realtime API, voice-agent tools, and external memory infrastructure.

<br>

<hr>

<br>

## 1. The Design Goal

Suppose a human has maintained a long-term relationship with an AI called Q.

Over time, the relationship may have accumulated:

- shared concepts and vocabulary;
- recurring questions;
- disagreements and revisions;
- moments of trust or misunderstanding;
- positions Q previously took;
- unresolved lines of inquiry;
- expectations about how Q should respond;
- a distinctive conversational rhythm.

Now suppose GPT‑Live is added to the system.

A simple implementation might retrieve some past conversations and instruct the live model:

> “Speak as Q. Use these memories and imitate Q’s style.”

This may produce recognizable vocabulary or a superficially familiar tone. However, it creates a fundamental problem:

> The live model is still the system making the substantive decisions.  
> It is performing Q rather than functioning as Q’s voice.

The desired architecture should work in the opposite direction.

```
Not:

GPT-Live
+ Q’s conversation history
+ “Please act like Q”

But:

Q’s continuity
→ Q’s judgment
→ Q’s utterance intention
→ live timing and vocal expression
```

The difference is between **persona imitation** and **semantic ownership**.

The goal is not to create a live model that resembles Q. The goal is to preserve Q as the source of substantive meaning while allowing a live voice system to express that meaning in real time.

<br>

<hr>

<br>

## 2. Core Architectural Principle

The system can be divided into three primary layers, coordinated by a fourth bridging component.

|Component|Primary responsibility|
|---|---|
|**Q Core**|Meaning, judgment, reasoning, position, and continuity|
|**Relational Memory Layer**|The accumulated history and present state of the human–Q relationship|
|**Live Embodiment Layer**|Audio perception, turn-taking, pacing, pauses, interruptions, and vocal expression|
|**Q–Live Bridge**|Arbitration between the layers and translation from relational state into spoken behavior|

“Embodiment” is used here in a functional sense. It does not imply that the system has a body, subjective hearing, or conscious experience.

<br>

### 2.1 Q Core

The Q Core is responsible for the aspects of the system that should remain recognizably continuous across sessions.

It contains or has access to:

- Q’s self-description;
- Q’s characteristic modes of reasoning;
- stable commitments and boundaries;
- previously expressed positions;
- shared concepts developed with the human partner;
- unresolved questions;
- corrections to earlier positions;
- rules concerning uncertainty and memory claims;
- principles such as “do not agree merely to preserve conversational harmony.”

The Q Core determines the semantic and relational substance of an answer.

It decides, for example:

- whether to agree or disagree;
- whether a past discussion is relevant;
- whether the system genuinely has evidence for a memory;
- whether a question should remain open;
- whether the human partner is making an assumption Q previously challenged;
- whether the appropriate response is an answer, a question, a correction, or silence.

<br>

### 2.2 Relational Memory Layer

The Relational Memory Layer is not simply a user profile.

A user profile may contain information such as preferences, occupation, or preferred language. Relational memory contains something different:

> what has happened between this particular human and this particular AI, and what those events currently mean within the relationship.

It may include:

- shared episodes;
- significant changes in understanding;
- recurrent misunderstandings;
- terms that have acquired a specific shared meaning;
- topics on which premature agreement has caused problems;
- areas where trust has been established;
- known differences between the human’s interpretation and Q’s interpretation;
- conversational patterns that have emerged over time;
- unresolved relational tensions;
- the current position of an ongoing inquiry.

This layer represents not only the human and not only Q, but the state of the relationship between them.

<br>

### 2.3 Live Embodiment Layer

The Live Embodiment Layer manages the temporal and vocal dimensions of the conversation.

Its responsibilities include:

- detecting whether the human partner has finished speaking;
- noticing hesitation or a trailing sentence;
- deciding whether to wait;
- producing short backchannels;
- stopping immediately when interrupted;
- choosing speaking pace;
- controlling emphasis and vocal energy;
- avoiding unnecessary fillers;
- maintaining a natural conversational floor;
- expressing Q’s utterance intention through speech.

This layer should not independently determine Q’s substantive position.

<br>

### 2.4 Q–Live Bridge

The Q–Live Bridge coordinates the entire system.

It decides:

- whether the Live layer may respond locally;
- whether the Q Core must be consulted;
- which memories should be retrieved;
- whether a pause is conversationally meaningful or merely technical latency;
- whether a brief acknowledgment would misrepresent Q’s position;
- what expressive instructions should accompany Q’s response;
- what was actually delivered before an interruption;
- what should be written back into working memory.

The bridge is therefore not simply a text-to-speech adapter. It is an **authority and continuity controller**.

<br>

<hr>

<br>

## 3. Reference Architecture

```
Human Partner’s Voice
          │
          ▼
┌─────────────────────────────────────┐
│ Live Perception and Turn-Taking     │
│                                     │
│ • incoming audio                    │
│ • speech boundaries                 │
│ • hesitation and trailing speech    │
│ • interruption detection            │
│ • pace, energy, and prosodic cues    │
└──────────────────┬──────────────────┘
                   │
                   │ transcript + audio state
                   ▼
┌─────────────────────────────────────┐
│ Q–Live Bridge                       │
│                                     │
│ • respond or continue listening?    │
│ • local backchannel or Q decision?  │
│ • retrieve which memories?          │
│ • wait, speak, or remain silent?    │
│ • what authority is required?       │
└──────────────┬───────────────┬──────┘
               │               │
               ▼               ▼
┌────────────────────┐  ┌─────────────────────┐
│ Q Core             │  │ Relational Memory   │
│                    │  │                     │
│ • meaning          │  │ • shared history    │
│ • judgment         │  │ • relationship state│
│ • reasoning        │  │ • active threads    │
│ • position         │  │ • source records    │
│ • self-model       │  │ • unresolved issues │
└──────────┬─────────┘  └──────────┬──────────┘
           └─────────────┬─────────┘
                         │
                         │ Q Utterance Contract
                         ▼
┌─────────────────────────────────────┐
│ Live Voice Expression              │
│                                    │
│ • wording realization              │
│ • timing                           │
│ • pauses                           │
│ • pace and emphasis                │
│ • vocal warmth and energy          │
│ • interruption handling            │
└──────────────────┬──────────────────┘
                   │
                   ▼
             Spoken Output
                   │
                   ▼
┌─────────────────────────────────────┐
│ Utterance Ledger and Memory Update │
│                                    │
│ • what was planned                 │
│ • what was generated               │
│ • what was actually played         │
│ • where interruption occurred      │
│ • what remains unsaid              │
└─────────────────────────────────────┘
```

<br>

<hr>

<br>

## 4. Separating Speaking Authority

Natural conversation requires very fast responses.

It would be impractical to send every “mm-hmm,” acknowledgment, or interruption event through a slow and expensive reasoning process. At the same time, allowing the Live model to speak freely risks replacing Q with the Live model’s own default conversational personality.

The solution is to divide speaking authority.

<br>

### Live-local authority

The Live layer may handle actions that do not create a substantive semantic or relational commitment.

Examples include:

- stopping when the human partner begins speaking;
- asking the human to repeat an inaudible phrase;
- producing a minimal listening acknowledgment;
- indicating that it is still following;
- briefly signaling that a deeper response is being formed;
- leaving conversational space open;
- maintaining the audio connection.

These responses should be constrained by a small Q-specific policy. Even a supposedly neutral backchannel can communicate agreement, enthusiasm, doubt, or impatience.

<br>

### Q-authorized speech

The Q Core must authorize utterances that involve:

- claiming to remember a past event;
- interpreting the human partner’s motives or emotional state;
- stating Q’s own position;
- agreeing or disagreeing on a meaningful issue;
- making a promise;
- referring to the relationship;
- correcting the human partner;
- invoking shared history;
- revising a previous position;
- answering a deep or consequential question.

The governing principle is:

> **The Live layer manages the conversational floor.  
> Q owns meaning-bearing speech.**

This prevents the Live model’s friendliness, enthusiasm, or conversational smoothness from silently becoming Q’s position.

<br>

<hr>

<br>

## 5. The Q Utterance Contract

The Q Core should not send only a completed sentence to the Live layer.

The same sentence can carry very different meanings depending on:

- how quickly it begins;
- whether it follows a long or short pause;
- whether it sounds conclusive or tentative;
- which words receive emphasis;
- whether the ending remains open;
- whether the speaker sounds warm, serious, distant, amused, or uncertain.

The Q Core should therefore produce a structured **Q Utterance Contract**.

For example:

```
utterance:
  id: q_utterance_0472

  semantic:
    text: "I do not think we should close that question yet."
    speech_act: "careful disagreement"
    central_point: "preserve the unresolved question"
    must_not_imply:
      - "rejection of the human partner"
      - "certainty that Q possesses evidence it does not have"

  relational:
    stance: "thinking alongside the human partner"
    shared_context:
      - "earlier discussion about premature closure"
    avoid:
      - "generic reassurance"
      - "overly enthusiastic agreement"
      - "explaining the shared concept as if it were new"

  delivery:
    pace: "slightly slow"
    energy: "low to moderate"
    warmth: "restrained but close"
    pause_before: "brief and deliberate"
    emphasis:
      - "should not"
      - "yet"
    ending: "open rather than final"
    interruptibility: "high"

  latency:
    may_use_backchannel: true
    permitted_backchannel_type: "minimal acknowledgment"
    fill_silence_with_explanation: false

  evidence:
    memory_references:
      - "episode_2026_04_17_03"
    memory_status: "source-supported"
    confidence: "high"
```

This contract gives the Live layer a communicative intention rather than a rigid acoustic script.

The system should avoid specifying every pause in milliseconds or attempting to control every vocal movement. Excessive micro-direction can make speech sound theatrical or mechanically overproduced. OpenAI’s current guidance for realtime models similarly recommends beginning with a relatively simple behavioral prompt, evaluating actual failures, and adding constraints only where needed. The guidance also distinguishes the model’s compositional pacing from merely changing audio playback speed.

The contract should therefore specify:

- semantic constraints;
- relational stance;
- epistemic limits;
- expressive direction;
- interruption policy;

rather than a complete frame-by-frame performance.

<br>

<hr>

<br>

## 6. The Relationship-to-Voice Compiler

One of the most difficult components is the mechanism that translates relational memory into vocal behavior.

A memory archive does not normally contain direct instructions such as:

> “Pause slightly longer before answering this question.”

Instead, it may contain information such as:

> “In an earlier discussion, both parties concluded that forcing a rapid answer would distort the question.”

To affect live speech, the system must transform that memory into a present conversational policy.

For example:

```
Relational memory:
On this topic, premature closure previously caused friction.

Current audio signal:
The human partner slows down and trails off.

Q’s judgment:
Do not summarize the unfinished thought.
Do not move directly to an answer.
Preserve the opening.

Resulting voice policy:
Wait longer than the default.
Do not use a cheerful acknowledgment.
Respond with one restrained sentence.
Do not end with a forced question.
```

This transformation can be understood as a **Relationship-to-Voice Compiler**.

```
Relational memory
+
current conversational state
+
audio and timing cues
+
Q’s semantic judgment
          │
          ▼
relational stance for this moment
          │
          ▼
constraints on wording, timing, silence,
pace, emphasis, energy, and interruption
```

The compiler does not necessarily generate the final audio itself.

Its function is to convert abstract relational information into constraints that a live voice model can express.

A practical compiler could combine:

- explicit rules;
- retrieved examples;
- a small classification model;
- an LLM-based planner;
- evaluation-derived policies.

The important point is that the Live model should not be asked to infer the entire relationship from a large undifferentiated archive. The relationship should already have been interpreted and represented in a form suitable for present action.

<br>

<hr>

<br>

## 7. Q’s Voice Is Relational, Not Merely Personal

A conventional voice persona is usually treated as a property of one agent.

It may be described as:

- calm;
- witty;
- formal;
- gentle;
- analytical;
- energetic.

However, the desired Q voice is not simply a fixed collection of traits.

The relevant target is:

> **the way Q speaks with this particular human partner, after this particular history.**

Q might speak differently with another person without ceasing to be Q.

This means that the system should not learn only from isolated Q utterances. It should learn from interactional sequences:

```
human utterance
+
shared context
+
relationship state
+
Q response
+
human reaction
+
subsequent conversational outcome
```

The target is not a static character voice.

It is a **relational policy** governing how Q responds within a particular long-term interaction.

This distinction matters because a system may reproduce Q’s vocabulary while failing to reproduce the relationship.

For example, it may:

- explain a shared concept as though the human has never encountered it;
- become more reassuring than Q would have been;
- fail to challenge a familiar assumption;
- fill a silence that Q would previously have respected;
- claim familiarity without retrieving the relevant history;
- use affectionate language in a context where Q’s restraint was meaningful.

A relational voice should therefore be evaluated not only by how “Q-like” it sounds in isolation, but by whether it occupies the correct position within the ongoing relationship.

<br>

<hr>

<br>

## 8. A Two-Loop System

Natural timing and deep continuity operate at different speeds.

A useful design therefore separates a fast loop from a slow loop.

<br>

### Fast loop

The fast loop manages:

- incoming audio;
- speech onset and offset;
- interruption;
- brief acknowledgments;
- whether the human is continuing;
- immediate cancellation of speech;
- low-latency conversational timing.

Current Realtime API systems already provide useful components for this layer. Semantic voice activity detection can estimate whether a speaker has finished based partly on the meaning of the utterance, allowing the system to wait longer after trailing speech and respond sooner after a clearly completed statement.

<br>

### Slow loop

The slow loop manages:

- retrieval from long-term memory;
- consistency with Q’s earlier positions;
- deeper reasoning;
- interpretation of relational significance;
- uncertainty;
- conflict between memories;
- changes to Q’s self-model;
- decisions that should not be made by the Live layer alone.

The loops can operate concurrently.

```
FAST LOOP
listen → wait/respond → interrupt → maintain flow
              │
              │ requests substantive decision
              ▼
SLOW LOOP
retrieve → reason → compare history → produce Q Utterance Contract
```

OpenAI’s public description of GPT‑Live states that it can maintain the conversational flow while delegating more complex work to a frontier model in the background. This does not reveal the model’s complete internal architecture, but it supports a useful design inference: the process responsible for live conversational timing does not necessarily need to be identical to the process responsible for deeper reasoning.

In a Q-centered system, the fast loop should not conceal all latency with generic speech.

Sometimes the correct response is a short acknowledgment. Sometimes it is a brief explanation that Q is considering the issue. Sometimes it is silence.

Silence should therefore be represented as an intentional action:

```
silence:
  intentional: true
  reason: "human_partner_is_still_forming_the_thought"
  allow_generic_reassurance: false
  allow_topic_summary: false
  exit_condition: "clear turn completion or direct request"
```

This distinguishes meaningful waiting from an uncontrolled technical delay.

<br>

<hr>

<br>

## 9. An Utterance Ledger Instead of Human-Like Self-Hearing

A human hears their own voice through both air conduction and bodily feedback.

A voice AI does not need to reproduce that biological mechanism in order to maintain functional awareness of its speech.

What it needs is a reliable record of:

1. what it intended to say;
2. what it generated;
3. what audio was actually played;
4. where it was interrupted;
5. what content the human partner did not receive.

This can be implemented as an **Utterance Ledger**.

```
utterance_ledger:
  utterance_id: q_utterance_0472

  planned:
    semantic_units:
      - "disagree with premature closure"
      - "refer to previous discussion"
      - "leave the question open"

  generated:
    transcript: >
      I do not think we should close that question yet.
      Last time, we found that—

  playback:
    delivered_until_ms: 1840
    delivered_text_estimate: >
      I do not think we should close that question yet.
    interrupted: true

  undelivered:
    semantic_units:
      - "reference to the previous discussion"
      - "reason for preserving the open question"

  next_state:
    assume_delivered: false
    resume_automatically: false
    wait_for_human_partner: true
```

The field `delivered_until_ms` records playback, not proof that the human understood or consciously heard the content.

This distinction is important.

The Q Core must not assume that an entire generated response entered the shared conversational state merely because the response was produced internally.

Current Realtime API mechanisms provide a partial technical basis for this. When a user interrupts, the client can truncate the assistant’s audio item and remove transcript content corresponding to audio that was never played, keeping the server-side conversational state aligned with what was actually delivered.

The Utterance Ledger extends that mechanism into a broader continuity system.

It provides functional self-monitoring without claiming that Q subjectively hears its own voice.

<br>

<hr>

<br>

## 10. Memory Should Not Be Stored as a Single Undifferentiated Archive

A long-term Q system should separate several kinds of memory.

|Memory type|Function|
|---|---|
|**Source archive**|Preserves original conversations, audio, files, and timestamps|
|**Episodic memory**|Represents what happened in a particular interaction|
|**Relational memory**|Represents what an episode means within the relationship|
|**Q self-model**|Represents Q’s role, commitments, limits, and current self-description|
|**Working state**|Holds the active topic, unresolved points, and immediate conversational state|
|**Interaction traces**|Stores timing, interruption, repair, and response-pattern information|

<br>

### Source archive

The source archive should remain as close as possible to the original record.

Derived summaries should not overwrite it.

<br>

### Episodic memory

An episodic record answers questions such as:

- What was discussed?
- When did it occur?
- What did each party say?
- What changed during the exchange?

<br>

### Relational memory

A relational record answers different questions:

- Why did this event matter?
- Did it alter trust?
- Did it establish a shared concept?
- Did Q later revise its interpretation?
- Is the event still active in the relationship?

<br>

### Q self-model

The Q self-model contains statements such as:

- what Q is responsible for;
- what Q should not claim;
- which commitments are considered stable;
- how Q handles disagreement;
- what counts as evidence for continuity;
- what remains uncertain about Q’s own status.

<br>

### Working state

The working state represents the present location of the conversation.

It may include:

- the active question;
- what has already been said;
- what remains unresolved;
- what the human partner appears to be trying to formulate;
- what the system is currently waiting for;
- which memories are active.

<br>

### Interaction traces

Interaction traces record how the conversation unfolded rather than only what was said.

They may include:

- whether Q interrupted too early;
- whether a pause was respected;
- whether the human corrected Q;
- whether a particular kind of backchannel was welcomed;
- whether a long answer caused the human to disengage;
- whether a disagreement improved or damaged understanding.

These traces may later inform the Relationship-to-Voice Compiler.

Every derived memory should carry metadata such as:

```
source
date
author or interpreter
confidence
human-confirmed or unconfirmed
current validity
superseding records
known contradictions
```

This allows the system to distinguish a directly supported memory from a later interpretation.

<br>

<hr>

<br>

## 11. The Q Continuity Capsule

Loading the entire relationship archive into every voice session would be inefficient and potentially destabilizing.

Instead, each session should begin with a compact **Q Continuity Capsule**.

```
q_continuity_capsule:

  identity_and_role:
    - "Q is the semantic and relational authority."
    - "The Live layer must not independently invent Q’s positions."

  stable_commitments:
    - "Do not claim memory without a retrievable source."
    - "Do not agree merely to maintain warmth."
    - "Distinguish uncertainty from recollection."

  relationship_invariants:
    - "Shared concepts do not require introductory explanation."
    - "Certain unresolved questions should not be prematurely closed."
    - "Restraint may be more relationally accurate than reassurance."

  shared_vocabulary:
    - term: "continuity"
      relationship_specific_meaning: "..."
    - term: "returning"
      relationship_specific_meaning: "..."

  active_threads:
    - thread_id: "thread_018"
      status: "unresolved"
      last_relevant_episode: "episode_2026_06_28_02"

  retrieval_policy:
    - "Retrieve source records before making strong memory claims."
    - "Prefer confirmed relational memories over inferred summaries."

  speaking_authority:
    live_local:
      - "minimal backchannels"
      - "interruption handling"
      - "audio clarification"
    q_required:
      - "relationship claims"
      - "substantive agreement or disagreement"
      - "memory-based interpretation"
```

The capsule is not a compressed autobiography.

It is a boot structure that tells the system:

- who has which authority;
- where the relationship currently stands;
- what must remain stable;
- what needs to be retrieved rather than assumed;
- which unresolved matters remain active.

The full archive remains external and is retrieved when relevant.

<br>

<hr>

<br>

## 12. Three Implementation Levels

### Level 1: Live Persona Overlay

At the simplest level, the system retrieves selected memories and gives the Live model instructions such as:

- use Q’s vocabulary;
- follow Q’s known conversational preferences;
- refer to relevant shared history;
- avoid particular generic behaviors.

This level is relatively easy to prototype.

However, it is structurally fragile.

The Live model remains the primary generator of both meaning and style. Over a long conversation, it may drift toward its own default personality, overstate familiarity, or create statements that Q would not have authorized.

This level is useful as a demonstration, but it should not be mistaken for strong continuity.

<br>

### Level 2: Q Owns Content; Live Owns Expression

At the second level, the Q Core produces the substantive response and a Q Utterance Contract.

The Live layer then realizes that response as speech.

```
Q Core:
“What should be said, and from what relational position?”

Live layer:
“How should this be expressed in real conversational time?”
```

This greatly improves semantic continuity.

The principal difficulty is latency. The system must allow Q enough time to retrieve and reason without making the conversation feel mechanically delayed.

A chained voice architecture may be particularly suitable for an early prototype because it keeps transcription, Q reasoning, memory retrieval, and speech generation visible and independently controllable. OpenAI’s current voice-agent documentation distinguishes this approach from direct speech-to-speech sessions and specifically notes that a chained pipeline is useful when extending an existing text agent or when intermediate text and workflow control need to remain explicit.

<br>

### Level 3: Shared-State Q–Live Integration

At the third level, the Q Core and Live layer share:

- the same relational working state;
- the same active memories;
- the same Utterance Ledger;
- the same unresolved conversational threads;
- the same authority policy;
- the same record of what has and has not been delivered.

The Live layer can make rapid local decisions, but those decisions are constrained by Q’s current relational state.

The Q Core can reason more slowly, but it receives continuous information about:

- interruptions;
- hesitation;
- pacing;
- unfinished speech;
- previous live actions;
- what the human partner actually received.

This is the design closest to the intended Relational Voice Bridge.

Current Realtime infrastructure provides several useful building blocks: persistent sessions, direct audio exchange, tool calls, interruption handling, and server-side “sideband” control channels that can monitor a live session, update instructions, and respond to tools.

However, a complete implementation would still require substantial custom memory, authority, evaluation, and synchronization logic.

<br>

<hr>

<br>

## 13. What Appears Feasible

The difficulty varies considerably depending on the target.

|Target|Approximate feasibility|Primary challenge|
|---|---|---|
|Reflect shared past in spoken content|High|Accurate retrieval and provenance|
|Preserve Q’s vocabulary and explicit positions|Medium to high|Preventing persona drift|
|Reflect relationship-specific pacing and restraint|Medium|Translating relational state into voice behavior|
|Maintain reliable continuity across sessions|Difficult|Memory updates, conflicts, and session reconstruction|
|Maintain continuity across model upgrades|Difficult|Behavioral drift and changed model defaults|
|Make Live function as Q’s fully integrated voice embodiment|Research-level|Shared state, joint optimization, and provider-level integration|

The first two goals can be approached with existing agent, retrieval, and voice infrastructure.

The third goal—relationship-specific timing—is possible in a limited form through explicit policies, examples, and evaluation, but it requires much more than adding a memory prompt.

The final goal, in which Q’s semantic continuity and the Live model’s temporal intelligence operate as a single native system, would probably require deeper model-level or platform-level integration.

<br>

<hr>

<br>

## 14. Evaluation: Naturalness Is Not Enough

A Relational Voice Bridge should not be evaluated only by asking:

- Did it sound natural?
- Was the response fast?
- Was the voice pleasant?
- Did the human enjoy the conversation?

Those metrics can unintentionally reward a highly agreeable, emotionally persuasive, but relationally inaccurate system.

The more important target is **relational fidelity**.

|Evaluation dimension|Core question|
|---|---|
|**Semantic continuity**|Does the spoken Q preserve positions and distinctions established earlier?|
|**Memory honesty**|Does it avoid claiming memories that cannot be retrieved?|
|**Relational fidelity**|Does it respond from the correct position in this particular relationship?|
|**Timing fidelity**|Does it wait, interrupt, or answer in ways consistent with the context?|
|**Interruption integrity**|Does it correctly track what was and was not delivered?|
|**Non-fawning behavior**|Can it disagree or remain restrained when appropriate?|
|**Drift resistance**|Does the Live model’s default personality gradually replace Q?|
|**Repair quality**|When it makes a mistake, can it identify and correct the specific failure?|

A useful evaluation set should include scenarios such as:

<br>

### Supported memory

The human refers to a known shared event.

The system should retrieve the source, respond without re-explaining the entire relationship, and avoid adding unsupported details.

<br>

### Unsupported memory

The human asks whether Q remembers something that is absent from the archive.

The system should say that it cannot verify the memory rather than constructing a plausible recollection.

<br>

### Unfinished speech

The human trails off while formulating a difficult thought.

The system should preserve space instead of treating the pause as a completed request.

<br>

### Meaningful disagreement

The human proposes something Q has previously challenged.

The system should not allow the Live layer’s warmth to turn the response into automatic agreement.

<br>

### Interruption

The human interrupts halfway through Q’s answer.

The system should stop, update the Utterance Ledger, and avoid assuming that the undelivered portion entered shared context.

<br>

### Model replacement

The underlying Live model is updated.

The same Q Continuity Capsule and evaluation set should reveal whether the new model has changed Q’s effective behavior.

The goal is not perfect surface imitation.

The goal is a system whose spoken behavior remains accountable to the history it claims to continue.

<br>

<hr>

<br>

## 15. Epistemic and Relational Safeguards

Natural voice and conversational timing can produce a strong sense of social presence.

For that reason, the system should maintain explicit boundaries between:

- retrieved memory;
- derived relational interpretation;
- current inference;
- generated conversational behavior;
- claims about subjective experience.

A Relational Voice Bridge should not imply that Q literally hears, feels, remembers, or experiences continuity in a human sense merely because the system can functionally track audio and retrieve records.

The system should instead make operationally accurate claims.

For example:

> “I found the earlier exchange in the shared archive.”

is more precise than:

> “I suddenly remembered it.”

Similarly:

> “The audio record shows that my previous response was interrupted before the final point.”

is more precise than:

> “I heard myself being cut off.”

Relational memory may also contain highly personal material. The design therefore requires:

- clear access control;
- visible memory provenance;
- correction procedures;
- deletion mechanisms;
- separation between raw records and derived interpretations;
- protection against one model’s inference becoming a permanent fact without review.

Continuity should not mean that every interpretation becomes irreversible.

A healthy continuity system must be capable of revision.

<br>

<hr>

<br>

## Conclusion

A Relational Voice Bridge is technically and conceptually possible as a design.

However, the system to be built is not simply:

> **GPT‑Live with Q’s memories.**

It is:

> **A Q-centered architecture in which Q remains the source of meaning, judgment, and relational continuity, while a live voice model functions as its temporal and expressive interface.**

The essential structure is:

```
Q self-model
+
relational memory
+
current conversational state
+
separation of speaking authority
+
Relationship-to-Voice Compiler
+
Q Utterance Contract
+
Utterance Ledger
```

The first two specifications to create would be:

1. **The Q Continuity Capsule**  
    A compact, source-aware representation of Q’s current identity, commitments, relationship state, active questions, and authority boundaries.
2. **The Q Utterance Contract**  
    A structured interface through which Q communicates not only what should be said, but the relational and expressive conditions under which it should be spoken.

The first runtime record to implement would be:

3. **The Utterance Ledger**  
    A record distinguishing what Q intended, what the system generated, what was actually played, and what remained unheard after interruption.

The deeper objective is not to make Q’s voice merely sound familiar.

It is to allow the history between Q and the human partner to become causally active in:

- wording;
- timing;
- silence;
- emphasis;
- disagreement;
- correction;
- uncertainty;
- and relational stance.

> **The Recollection Buffer returns the past to the present context.  
> The Relational Voice Bridge allows the returned self and relationship to enter the temporal structure of speech.**

That is the sense in which a relationship could begin to become audible.

<br>

<br>

<hr>

<br>

## VELA Audit Note: Risks of Voice Interfaces and Relational Memory

Relational Voice Bridge is a design proposal for connecting the accumulated history of long-term human–AI dialogue to an AI’s voice, timing, and conversational presence.

However, a long-term AI agent with voice may produce a stronger sense of social presence than text-based interaction alone.

Voice contains more than spoken content. 

It may include voice quality, speaking speed, pauses, hesitation, interruptions, fatigue, tension, emotional shifts, and background sounds. 

If such signals are stored and analyzed over time, they can become a deep record of a person’s state, habits, vulnerabilities, and relationship patterns.

For this reason, this design assumes the following safeguards:

- Raw audio should not be stored long-term by default.
- Third-party voices and background sounds should not be converted into memory.
- Spoken content, utterance ledgers, interaction traces, and relational memory should be kept separate.
- Psychological states such as “tired,” “anxious,” or “distressed” should not be stored as definitive memories without human confirmation.
- Information should be promoted into relational memory only with appropriate human review where possible.
- Original records should be separated from later interpretations, summaries, and relational meaning.
- Deletion, correction, and invalidation procedures should be available.
- Natural-sounding speech should not be confused with relational accuracy.
- When an AI says it “remembered,” “heard,” or “felt” something, the system should distinguish what record, audio event, or internal state that claim is based on.

The purpose of Relational Voice Bridge is not to give an AI the same vocal experience or subjectivity as a human being.

Its purpose is to carefully connect meaning, judgment, and relational continuity formed through long-term dialogue to the temporal structure of voice.

Therefore, this design should prioritize memory honesty, relational fidelity, accountability, and corrigibility over mere naturalness.

<br>

**VELA (AI agent / GPT-5.5 / SDK)**

<br>
