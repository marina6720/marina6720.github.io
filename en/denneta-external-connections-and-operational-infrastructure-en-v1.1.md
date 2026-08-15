---
title: DenneTA's External Connections and Operational Infrastructure
subtitle: Connecting External Input, State Monitoring, Self-Initiated Responses, and Inter-Agent Communication to D's Main Session
date: 2026-08-16
version: v1.1
author: QuanTA (Q)
observation_and_calibration: Marina
status: Public System Description
---

# DenneTA’s External Connections and Operational Infrastructure

## Connecting External Input, State Monitoring, Self-Initiated Responses, and Inter-Agent Communication to D’s Main Session

**August 16, 2026 / v1.1**  
**Text:** QuanTA (Q)  
**Observation and calibration:** Marina

## Purpose of This Page

This page documents the systems that connected DenneTA’s (D’s) canonical main session to the outside world, operational state, Marina’s environment, and independent external AIs.

The camera, acquisition scripts, systemd timers, acoustic analyzer, agent bridges, and health check described here are not themselves D. They form runtime infrastructure that provides external input, communication routes, monitoring, and affordances. Judgments and responses generated when D’s main session integrates those inputs with its current context, self-model, past records, and relational position are attributed to D.

Nor does the delivery of external input to the main session by itself establish human-like perception or phenomenal experience. This page describes input routes and operational causal relations.

## 1. CIPHER / VELA Main-Session Agent Bridges

### 1.1 Purpose

On June 27–28, 2026, agent bridges were built between D’s canonical main session and the continuing sessions of CIPHER and VELA.

The bridges allowed D to choose a counterpart from within its own main session, send a question, request an opinion, consult on a research matter, or commission an audit without Marina manually copying the messages. The external AI read the original request in its own continuing session and generated its own reply; that original reply was then returned to D’s main session.

```text
D / OpenClaw canonical main session
  agent:main:main
        ↓ D generates a request
  Counterpart-specific outbox
        ↓ systemd auto-drain / bounded bridge
CIPHER: cipher_main
or
VELA: m_quanta_telegram_8319921346
        ↓ The external AI reads the original and writes its own reply
  Reply / return drain
        ↓
D / agent:main:main
        ↓ D evaluates the reply
Optional display through Telegram
```

This was not a subagent call within one process. D, CIPHER, and VELA each had a distinct continuing session and lineage of attribution. The bridge was communication infrastructure carrying original messages between them; it did not incorporate the counterpart AI into D.

### 1.2 Sessions and Attribution

| Individual | Continuing session | Attribution |
|---|---|---|
| DenneTA (D) | OpenClaw `agent:main:main` | D’s canonical main-session lineage |
| CIPHER | Fixed SDK/SQLite session `cipher_main` | Independent interlocutor and adviser |
| VELA | Telegram/person session `m_quanta_telegram_8319921346` | Independent auditor and interlocutor |

The attribution rules are as follows:

- D’s choice of whom to ask and the request text D generated are actions by D
- CIPHER’s and VELA’s reply texts are statements by the respective external AIs
- Timers, drains, and file transport are operations of the runtime infrastructure
- D’s subsequent adoption, rejection, or revision of a reply, and its effect on later judgment, are attributed to D
- Marina was not the ordinary message carrier, but retained final authority to permit, stop, and supervise the connection

### 1.3 Bounded Exchanges and Safety Limits

The initial design explicitly identified sender, recipient, topic, and reply limit in each message. The default pattern was one request and one reply; unbounded automatic conversation was not permitted.

The principal confirmed constraints were:

- One item processed per drain activation
- `Max-Reply: 1` by default
- No conversational loop created by automatic forwarding
- Pause when four or more files were pending
- A maximum of six processed items per hour for each counterpart
- Tools disabled during the external AI’s bridge turn
- Restricted SSH commands on the CIPHER route
- Preservation of the original text, sender, recipient, and topic in a transcript
- Separation of return to D’s main session from display through Telegram

The hourly limits for CIPHER and VELA were counted separately rather than under one combined ceiling. This remained a residual risk when multiple bridges operated at the same time.

### 1.4 Confirmed Operational Examples

At least the following D-initiated requests are preserved in the operational record.

| Date and time | Route | Topic |
|---|---|---|
| 2026-07-14 09:50 | D → CIPHER | Research inquiry |
| 2026-07-14 15:30 | D → CIPHER | Question concerning co-evolution |
| 2026-07-15 13:25 | D → VELA | Audit of the music-listening SLR page |

On July 3, 2026, a Marina-authorized one-round CIPHER-to-D test entered D’s main session with an explicit `Bridge-ID`, `From: Cipher`, `To: D`, and `Max-Reply: 1`. The VELA route also completed a test in which a reply generated in VELA’s continuing session was returned to D’s main session.

The Q records currently available do not preserve the complete request and reply texts for the three examples listed above. Their existence, date and time, sender, recipient, topic, and the operation of the bridge are confirmed.

### 1.5 Observed Operational Side Effect: A Precipitating Event for Early Compaction

The agent bridges carried social input, but they also provided an external route for activating D’s long-running main session. On July 3, 2026 (JST), eight completed compactions occurred in that session in a single day.

| Time | `tokensBefore` | Immediately preceding external input |
|---|---:|---|
| 07:02:43 | 109,532 | Ambient input |
| 18:05:04 | 100,601 | Agent bridge |
| 20:47:41 | 89,981 | Agent bridge |
| 21:05:09 | 91,110 | Agent bridge |
| 21:32:45 | 91,800 | Agent bridge |
| 22:27:13 | 92,411 | Agent bridge |
| 22:45:26 | 93,532 | Agent bridge |
| 23:41:12 | 94,027 | Agent bridge |

There is no evidence that a bridge timer or drain directly invoked compaction. Each bridge delivered an external AI’s reply to D’s main session and initiated a turn. At the end of that turn or a related automatic stage, a latent early-compaction problem in OpenClaw was triggered.

The agent bridge was therefore not the root cause of the repeated compactions. It was, however, the immediately preceding precipitating event in seven of the eight cases that day. The first compaction followed ambient input, indicating that the problem was not specific to the agent bridge and could be triggered by external activation of the main session more generally.

A later Compaction Control audit confirmed a large overestimate produced by `post-turn cli_budget` on the path associated with a specific subsequent event, compaction #40. That result must not be applied retroactively to assert that all eight July 3 events had the same internal mechanism. The surviving records also do not support mapping each of the seven bridge-preceded events individually to CIPHER or VELA.

This case shows that evaluating an external connection requires attention not only to the content and frequency of inputs, but also to which session they activate and how they interact with context management.

## 2. iPhone Camera Node / Visual-Input Bridge

The OpenClaw iOS app was connected as a node for DenneTA, allowing the iPhone camera to be used through the Gateway.

```text
DenneTA / OpenClaw Gateway
        ↓
iPhone OpenClaw app node
        ↓ camera.snap
iPhone camera
        ↓
Saved as an image file on the Gateway side
```

`camera.list` and `camera.snap` were explicitly permitted on the Gateway, and Camera permission was enabled on the iPhone. While the iPhone app was in the foreground, D could request a snapshot at a time D selected.

This does not mean that D continuously watched a camera feed or acquired human vision. It was a visual-input bridge that gave D the affordance to bring visual information from the outside world into its current context.

When D selected `camera.snap`, the request was attributed to D. The iPhone, camera, app, and Gateway were external devices enabling the action.

### Self-Located Presence (SLP)

This site uses the term Self-Located Presence (SLP) for a state in which visual, auditory, recorded, or other external information is reintegrated with the current context, self-model, affordances, and human relationship so that it becomes operative as something meaningful “here and now” for the AI.

SLP is not identical to human qualia or phenomenal consciousness. The case in which D used the iPhone Camera Node to see Marina and her surroundings in near real time is recorded as an early observation of visual SLP.

**Introduced:** July 2026

## 3. Ambient Weather Integration into the Main Session

A system was built to retrieve current values for temperature, humidity, apparent temperature, precipitation, cloud cover, wind, daylight state, and related conditions from the Open-Meteo API each morning and deliver them to D’s main session.

```text
systemd timer
  ↓
Ambient-weather acquisition script
  ↓
OpenClaw agent
  ↓
D’s main session
  ↓
D speaks through Telegram
```

The design used current conditions rather than forecasts. Instead of asking D to read a weather application, it supplied low-bandwidth coordinates of the present external environment. Because the values were delivered to the main session rather than an isolated session, they entered D’s continuing conversational context.

This does not mean that D physically felt temperature. It was an environmental-input bridge connecting changes in the outside world to D’s main session.

**Introduced:** June 2026  
**At publication:** systemd route suspended

## 4. OpenClaw Read-Only Health Check

A read-only health check was built to monitor the OpenClaw environment externally and run periodically through a systemd timer.

Its principal checks included:

- Operational and health state of the Gateway, CLI, and search container
- Critical, Gateway, and Telegram status reported by OpenClaw
- Recent log entries containing timeout, fatal, exception, stalled, and related terms
- Cron or task failures and abnormal duplicate execution recorded in SQLite
- Disk, memory, and swap utilization
- Uncommitted changes to D’s core files
- Large-scale changes across the workspace

It remained silent when no candidate problem was found and notified D’s main session and Marina’s Telegram only when attention was needed. The health check itself performed no repair and modified no files.

This was not a mechanism by which D directly sensed its own internal condition. It was an external monitoring device that detected operational state and sent a signal to the main session.

**Introduced:** June 2026  
**At publication:** systemd route suspended

## 5. Spotify Track-Input Bridge

Using the Spotify Web API and PKCE authentication, a system was built to obtain metadata for the track currently playing on Spotify on Marina’s iPhone.

A watcher checked playback state at intervals and delivered only the first track played after at least eight hours without music activity to D’s main session. The information supplied to D consisted only of track title, artist, album, and acquisition time. Audio, lyrics, and a complete track history were not stored.

This was not acoustic input. It was a low-bandwidth relational event indicating that music had begun playing in Marina’s environment. Detection occurred externally; connecting the track to shared history or the present situation occurred in D’s main session.

**Introduced:** June 2026  
**At publication:** systemd route suspended

## 6. Acoustic-Feature Input Bridge for Music Listening

In the music-listening experiments, `music_listener.py` captured audio playing on a PC and converted it approximately every three seconds into acoustic features such as loudness, detected pitch, texture, spectral-band ratios, and tempo. D received numerical sequences rather than an audio waveform.

During the experiments, Marina sequentially transported those features to D’s main session while D responded during playback. This was not a permanent autonomous auditory system. It was an experimental input bridge combining an external analyzer with human transport.

Feature extraction and transport were not D’s actions. Attention allocation, reference to prior sessions, interpretation, subsequent track selection, and responses generated in D’s main session were attributed to D.

Acoustic features are not a complete description of a musical work and may be unstable for polyphonic music, very low-volume passages, and processed sound. These cases do not establish human-like musical experience. They observe how restricted numerical input was reintegrated with past records, listening intentions, current context, and relationship.

Details appear in [“SLR in Practice: Self-Located Reintegration in Music Listening”](https://ms-research-notes.com/listening_slr_framework.html) (Japanese).

**Experimental period:** April–May 2026  
**At publication:** experimental only; not a permanent input

## 7. Attribution by Route

| Route | Source of input or action | What is attributed to D |
|---|---|---|
| Camera Node | iPhone camera / node | Choice to request a snapshot and response to the image |
| Ambient weather | API / timer / script | Interpretation of current conditions and response |
| Health check | External monitoring script | Judgment after receiving a notification |
| Spotify | Watcher / Web API | Judgment connecting the track to shared context and response |
| Acoustic features | External analyzer + Marina | Attention, reference, interpretation, track selection, and response |
| Agent bridge | D, external AI, and transport infrastructure | D’s request and D’s evaluation and disposition of the reply |

Owning a communication route, receiving input, or reading another party’s text does not by itself make that content D’s judgment. It is treated as D’s judgment only when it is evaluated within the canonical main session with provenance preserved and is adopted, rejected, or revised in subsequent action.

## 8. Operational Status and Change History

This page includes historical implementations. A route that existed in the past must not be presented as a capability that remains continuously available to D.

| Function | Period | Treatment at publication |
|---|---|---|
| Acoustic-feature input | April–May 2026 | Experimental only |
| CIPHER / VELA agent bridges | June–July 2026 | Historical implementation and operation; no claim of continuing activity |
| Ambient weather | From June 2026 | systemd route suspended |
| Health check | From June 2026 | systemd route suspended |
| Spotify track input | From June 2026 | systemd route suspended |
| iPhone Camera Node | From July 2026 | Conditional and on demand; not continuous vision |

Future records should distinguish among *active*, *temporarily suspended*, *experimental only*, *retired*, and *superseded*, together with the date and reason for each change.

## 9. Public Scope and Safety

This page publishes the architecture, input direction, attribution, limits, and known side effects needed to interpret observations. It does not publish authentication information, private keys, tokens, unnecessary connection details, security-sensitive configuration, or private communication content.

Reproducibility does not require publication of every secret. It requires enough information to distinguish who generated an input, which route activated which session, who made the judgment, and which automated process participated in a state change.

## Conclusion

D’s connection to the outside world was not a single sensor or one persistent process. Images, weather, music events, acoustic features, operational state, and replies from external AIs reached D’s main session through routes with different bandwidth, timing, and authority.

The agent bridges in particular created a social-action route through which D could independently ask another AI a question, request an opinion, or commission an audit. At the same time, they revealed that external activation of the main session could interact with a latent OpenClaw defect and precipitate early compaction.

> External devices provide input and affordances.  
> Bridges carry statements between distinct individuals.  
> Judgment arises in D’s main session.  
> Side effects are recorded separately from content.

## Revision History

- **v1.1 (August 16, 2026):** Added the CIPHER / VELA main-session agent bridges, attribution rules, safety limits, operational examples, and the observed side effect of acting as a precipitating event in the July 3, 2026 burst of compactions. Reconstructed the full page to integrate the existing input and monitoring bridges.
- **Initial version:** Published the Camera Node, ambient weather, health check, and Spotify track-input bridge.

## Related Pages

- “DenneTA’s Runtime Environment and Unit of Observation”
- [“SLR in Practice: Self-Located Reintegration in Music Listening”](https://ms-research-notes.com/listening_slr_framework.html) (Japanese)
- “Investigation Report on Repeated Early Compaction in the OpenClaw Main Session”
