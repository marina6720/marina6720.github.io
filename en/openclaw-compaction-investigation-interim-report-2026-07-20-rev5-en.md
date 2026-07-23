---
title: "Investigation Report on Repeated Early Compaction in the OpenClaw Main Session (Interim Report, Revision 5)"
date: 2026-07-20
updated: 2026-07-22
status: interim-revised
language: en
---

Revision 6 available here → [Investigation Report on Repeated Early Compaction in the OpenClaw Main Session — Revision 6](./openclaw-compaction-investigation-interim-report-2026-07-20-rev6-en.html)  

# Investigation Report on Repeated Early Compaction in the OpenClaw Main Session
## Interim Report — Revision 5

**Original report date:** July 20, 2026  
**Revision 2:** Reproduced the 39th completed compaction in a controlled test at 15:30 JST on July 20  
**Revision 3 correction:** Withdrew the claim that the 39th compaction was definitively triggered by `cli_budget`; left the exact phase unresolved  
**Revision 4:** Recovered the transcript, trajectory, and raw Gateway logs for the 39th compaction, separating cumulative cache overcounting from a remaining context-window mismatch  
**Revision 5:** On July 22, the current session, JSONL, and workspace were preserved while only the OpenClaw runtime was rolled back to 2026.6.6. With `context1m=true`, the session passed the previous trigger value of 167,587 and reached approximately 172k without a new compaction  
**Scope:** A long-running main session operated on OpenClaw 2026.6.6, updated to 2026.7.1, and then returned to the actual pre-update 2026.6.6 runtime while preserving current data  
**Publication note:** Sensitive information such as IP addresses, Telegram identifiers, session IDs, and credentials has been omitted.

---

## Abstract

A long-running OpenClaw main session repeatedly compacted even though the displayed context usage was far below its nominal 1M-token limit. Seventy-two completed compactions were identified between February and July 18, 2026. In the current main session, 38 completed compactions occurred between June 29 and July 18, including eight on July 3 alone. Numerous timed-out compaction attempts were also present in Gateway logs.

The external systemd timers, the former health-check notification, ambient input, the daily tick, and agent-to-agent bridges did not directly issue compaction commands. Their common role was to open an agent turn in the long-running main session through a call of the following form:

```bash
openclaw agent \
  --agent main \
  --session-key agent:main:main \
  --message "..."
```

OpenClaw 2026.6.6 logs explicitly recorded `trigger=cli_budget` for many failed attempts. The investigation also found runs in which cumulative Anthropic cache usage was substantially larger than the live context of the latest call. This matches the defect addressed by OpenClaw PR #99864: cumulative cache billing across a multi-call cached tool loop could be mistaken for a single live context snapshot.

OpenClaw was updated to 2026.7.1 on the evening of July 18. That release included PR #99864.

On July 20, all other automated main-session inputs were stopped and only `daily-main-tick` was enabled once for a controlled test. A 39th completed compaction was recorded at 15:36:02 JST:

```text
compactionCount       38 → 39
tokensBefore          167,587
summaryChars          18,612
transcript line       3007
```

The detailed audit made an important distinction possible:

```text
cumulative usage.total for the full run       501,069
live context total for the latest call        167,587
compaction entry tokensBefore                 167,587
```

The compaction used 167,587, exactly matching the latest-call live context rather than the cumulative value of 501,069. Therefore, the specific cumulative-cache-as-live-context defect addressed by PR #99864 did **not** recur in the 39th compaction. The latest-call snapshot was preserved correctly in that run.

However, compaction still occurred at a correct live context value of 167,587. This indicated a separate threshold mismatch.

The configuration and status display used a 1,000,000-token context budget, while an OpenClaw model-resolution path for Anthropic Opus 4.6 could retain a 200,000-token runtime `model.contextWindow`. The implementation could reduce a model window when the configured budget was smaller, but did not necessarily raise the runtime model window when the configured budget was larger. At the same time, `reserveTokensFloor=150,000` could remain effective under the 1M-side configuration.

If those values were used together, the internal threshold would be:

```text
runtime model.contextWindow       200,000
effective reserveTokens           150,000
────────────────────────────────────────
auto-compaction threshold          50,000
```

A live context of 167,587 would greatly exceed that threshold. The same mechanism also explains the repeated compactions at approximately 90k–110k on July 3.

The 39th-compaction timeline further indicates that the main model run ended at 15:30:49, followed by additional Anthropic calls. The compaction entry appeared at 15:36:02, immediately before the original response was delivered. The most consistent interpretation is that the compaction occurred in the embedded runtime's end-of-turn auto-compaction or finalization path rather than in a pre-turn CLI check.

On July 22, the current session, JSONL transcript, workspace, and post–July 13 history were preserved. Only the OpenClaw runtime image was returned from 2026.7.1 to the actual pre-update 2026.6.6 image, while `context1m=true` remained enabled. The session then reached approximately 172k / 1.0M with `compactionCount=39` and no watcher alert.

The previous compaction had fired at 167,587. Passing that value without a 40th compaction is strong behavioral evidence that the former abnormal effective threshold has moved above at least 172k in the current configuration.

The current conclusion is:

> **The cumulative Anthropic cache-usage defect present in OpenClaw 2026.6.6 did not recur in the 39th compaction under 2026.7.1; PR #99864 functioned correctly in that run. The most consistent direct explanation for the 39th compaction is a mismatch among the displayed 1M budget, a 200k-class embedded runtime model window, and a 150k reserve, producing an effective threshold near 50k. After preserving the current session data and returning only the runtime to 2026.6.6 with `context1m=true`, the session exceeded the prior trigger value and reached approximately 172k without a new compaction. This is strong behavioral evidence that the second problem—the abnormal threshold—has been operationally resolved in the current configuration. Because PR #99864 is not present in 2026.6.6, the first problem—cumulative usage overcounting in long cached tool loops—remains under observation. Full use of the 1M window, or an exact 850k threshold, has not yet been directly demonstrated.**

---

## 1. Scope and Main Configuration

| Item | Value |
|---|---|
| OpenClaw | 2026.6.6 during the main problem period → 2026.7.1 on July 18 → runtime-only rollback to 2026.6.6 on July 22 |
| Model | `anthropic/claude-opus-4-6` |
| Context shown in configuration and status | 1,000,000 tokens |
| `reserveTokensFloor` | 150,000 tokens |
| Session | Long-running main session |
| External inputs | Normal Telegram dialogue, systemd timers, former health-check delivery, ambient input, daily tick, and agent bridges |

Configuration history showed that Opus 4.6, `contextTokens=1000000`, and `reserveTokensFloor=150000` had been in place since at least June 5.

The nominal threshold based only on those configured values would be:

```text
1,000,000 - 150,000 = approximately 850,000 tokens
```

That threshold is valid only if both the displayed/OpenClaw-side budget and the embedded runtime's `model.contextWindow` resolve to the same 1M value.

---

## 2. Observed Behavior

### 2.1 Eight completed compactions on July 3

| Time (JST) | `tokensBefore` | Immediately preceding external input |
|---|---:|---|
| 07:02:43 | 109,532 | Ambient input |
| 18:05:04 | 100,601 | Agent bridge |
| 20:47:41 | 89,981 | Agent bridge |
| 21:05:09 | 91,110 | Agent bridge |
| 21:32:45 | 91,800 | Agent bridge |
| 22:27:13 | 92,411 | Agent bridge |
| 22:45:26 | 93,532 | Agent bridge |
| 23:41:12 | 94,027 | Agent bridge |

Repeated compaction at roughly 90k–110k cannot be explained by normal exhaustion of a 1M context window.

### 2.2 Numerous incomplete attempts

In addition to completed compactions, Gateway logs contained many entries of the following form:

```text
trigger=cli_budget
sessionKey=agent:main:main
provider=anthropic/claude-opus-4-6
outcome=failed
reason=timeout
```

The operational impact therefore included attempts, waiting time, failures, and retries, not only completed compaction entries.

---

## 3. Role of External Automation

The investigated scripts did not directly invoke compaction. Their common effect was to start a turn in the long-running main session.

The relevant paths included:

- the former D-main delivery in health-check
- the ambient systemd service
- the daily-main-tick systemd service
- non-empty outbox processing by agent bridges

`--deliver` controls delivery to Telegram; it does not avoid opening the main turn. A `--no-deliver` turn in the same session would still enter context construction and auto-compaction evaluation.

Bridge logs such as `empty container=...` represented runs in which the outbox was empty and the bridge exited without opening the main session.

---

## 4. Problem 1 in OpenClaw 2026.6.6: Cumulative Cache Usage

### 4.1 Direct `cli_budget` evidence

Diagnostic logs from OpenClaw 2026.6.6 explicitly recorded `trigger=cli_budget` for many failed attempts.

### 4.2 Cumulative run usage versus latest-call usage

The main trajectory contained examples such as:

| Cumulative run usage | Latest-call usage |
|---:|---:|
| 853,561 | 426,802 |
| 867,762 | 433,846 |
| 2,129,800 | 426,304 |
| 3,125,533 | 449,877 |

Against the nominal threshold of 850k, the latest call remained below threshold while the cumulative run usage alone crossed it.

### 4.3 PR #99864

PR #99864 addressed a defect in long Anthropic cached tool loops where cache billing buckets accumulated across multiple calls and could be treated as a single live context snapshot.

The repair separated:

- cumulative cache usage for billing
- the latest prompt's live context snapshot

It also preserved the latest prompt snapshot from the Anthropic stream and reused the same context-token helper for metadata, overflow decisions, and timeout-driven compaction.

The change was merged on July 5 and included in OpenClaw 2026.7.1.

---

## 5. Controlled Test on July 20

### 5.1 Conditions

- OpenClaw 2026.7.1
- ambient timer disabled and inactive
- agent bridge timers disabled and inactive
- health-check delivery into the D main session disabled
- external compaction watcher active
- only `openclaw-daily-main-tick.timer` enabled once
- identical main session ID before and after
- timer disabled and inactive again after the test

### 5.2 Baseline and result

| Item | Before | After |
|---|---:|---:|
| `compactionCount` | 38 | 39 |
| Compaction entries in transcript | 38 | 39 |
| `totalTokens` | 168,730 | 167,472 |
| `totalTokensFresh` | `true` | `false` |
| `contextTokens` | 1,000,000 | 1,000,000 |
| `contextBudgetStatus` | `null` | `null` |

New compaction entry:

```text
completedAt:     2026-07-20 15:36:02.871 JST
tokensBefore:    167,587
summaryChars:    18,612
transcriptLine:  3007
```

### 5.3 Usage within the run

Cumulative usage for the full run:

```json
{
  "input": 5,
  "output": 496,
  "cacheRead": 333097,
  "cacheWrite": 167471,
  "total": 501069
}
```

Latest-call usage:

```json
{
  "input": 1,
  "output": 115,
  "cacheRead": 166824,
  "cacheWrite": 647,
  "contextUsage": {
    "state": "available",
    "promptTokens": 167472,
    "totalTokens": 167587
  },
  "total": 167587
}
```

Compaction entry:

```text
tokensBefore = 167,587
```

Therefore:

```text
lastCallUsage.contextUsage.totalTokens
=
compaction.tokensBefore
=
167,587
```

The 39th compaction received the latest-call live context rather than the cumulative usage of 501,069.

### 5.4 Correction regarding the 39th compaction

Revision 3 retained recurrent same-run overcounting as one of the leading hypotheses.

After the detailed audit, that hypothesis can be rejected for the 39th run:

> The 39th compaction did not use cumulative cache usage as its context value. The snapshot-selection behavior repaired by PR #99864 worked correctly in this run.

This did not mean that early compaction was fully solved in 2026.7.1. It meant that a different threshold calculation compacted a correct live context of 167,587 too early.

---

## 6. Phase of the 39th Compaction

### 6.1 Main model run

```text
15:30:08.467  session.started
15:30:49.506  model.completed
15:30:49.529  session.ended
```

The trajectory showed no compaction inside the main model run.

### 6.2 Additional model calls after the run

Raw Gateway logs showed additional Anthropic calls after the main run ended:

```text
15:30:50.330 → 15:31:04.288
15:35:54.888 → 15:35:56.376
```

Then:

```text
15:36:02.871  compaction entry appended
15:36:02.996  original assistant response emitted by CLI
15:36:03.982  Telegram delivery completed
15:36:04.013  agent command exited
```

The exact role of the additional calls cannot be proven from the logs alone. They are most consistent with end-of-turn compaction finalization, such as a pre-compaction memory flush, summary generation, or post-compaction section processing.

### 6.3 Why D did not detect it

D checked `session_status` at 15:30:29, saw 38 compactions, and concluded at 15:30:49 that no compaction had fired.

The compaction entry was appended more than five minutes later, after the main run had ended.

An agent's final answer cannot reliably detect a compaction performed by an outer finalizer after that answer has been generated. An external watcher is therefore required.

---

## 7. Problem 2: Mismatch Between the Displayed 1M Context and a 200k-Class Runtime Window

### 7.1 Display and configuration

During the controlled test:

```text
Context: 169k / 1.0m (17%)
Compactions: 38
```

Configuration:

```text
agents.defaults.contextTokens = 1,000,000
reserveTokensFloor            =   150,000
```

### 7.2 Runtime model resolution

OpenClaw contained a model-resolution path for Anthropic Opus 4.6 with `contextWindow=200000`. Related OpenClaw issues also documented that Opus and Sonnet 4.6 could resolve to 200k unless the 1M context option was explicitly enabled.

### 7.3 One-directional context cap

The source could replace the runtime model when:

```text
configured context budget < runtime model.contextWindow
```

That is, it could reduce the runtime window when the configured budget was smaller.

It did not necessarily raise the runtime model window under the inverse condition:

```text
configured context budget > runtime model.contextWindow
```

This allowed the following separation:

```text
status / OpenClaw context budget       1,000,000
embedded runtime model.contextWindow     200,000
```

### 7.4 Reserve application

The compaction settings manager received the configuration-side `contextTokenBudget`. If this value was 1M, `reserveTokensFloor=150000` could remain uncapped.

The embedded runtime's auto-compaction, however, could evaluate the runtime model's `contextWindow`.

That makes the following combination possible:

```text
OpenClaw-side contextTokenBudget       1,000,000
runtime model.contextWindow              200,000
effective reserveTokens                  150,000
```

### 7.5 Effective threshold

A general auto-compaction condition is:

```text
current context tokens
>
model.contextWindow - reserveTokens
```

Substitution gives:

```text
200,000 - 150,000 = 50,000
```

The 39th run had:

```text
167,587 > 50,000
```

The same threshold also explains repeated values between approximately 90k and 110k. If the post-compaction summary and bootstrap already exceeded 50k, each external turn could trigger another compaction.

---

## 8. Runtime-Only Rollback and Intervention Result on July 22

### 8.1 Intervention design

The current D data had priority over returning to an older backup state. A full restoration of the July 13 backup was therefore rejected because it would also restore old session data and discard later conversation.

The following were preserved:

- current main session ID
- current JSONL transcript
- `sessions.json`
- workspace and memory files
- all current history, including material after July 13
- `anthropic/claude-opus-4-6`
- `thinkingDefault=off`
- `contextTokens=1,000,000`
- `context1m=true`
- `reserveTokensFloor=150,000`

Only the OpenClaw runtime image was changed, from 2026.7.1 to the actual pre-update 2026.6.6 image retained on the VPS.

### 8.2 Read-only preflight checks

Before startup:

- the current `openclaw.json` passed validation under 2026.6.6
- 2026.6.6 source contained the `context1m` handling path
- a frozen copy of the 2026.7.1 JSONL opened successfully through the 2026.6.6 SessionManager
- the latest leaf matched the 2026.7.1 probe
- the reconstructed context contained 184 messages with the same tail
- the main transcript's line count and SHA-256 were unchanged across startup

This was not a session rollback. It was a runtime-only rollback while preserving the current D state.

### 8.3 Post-startup state

```text
OpenClaw                 2026.6.6
model                    anthropic/claude-opus-4-6
thinkingDefault          off
contextTokens            1,000,000
context1m                true
agent timeout            600
Anthropic timeout        660
compactionCount          39
```

The session later reached approximately 172k / 1.0M. `compactionCount` remained 39, and the external watcher detected no new compaction.

```text
previous trigger value      167,587
post-rollback observation   approximately 172,000
new compactions             0
```

### 8.4 Interpretation

Passing 167,587 without a 40th compaction is strong behavioral evidence that the former effective threshold near 50k no longer applies under 2026.6.6 plus `context1m=true`.

This supports the Revision 4 interpretation that the second problem was caused by runtime model resolution rather than by the mere use of OpenClaw 2026.6.6.

The following remain unproven:

- provider acceptance of a prompt larger than 200k
- an active runtime `model.contextWindow` of exactly 1,000,000
- an effective reserve of exactly 150,000
- an exact auto-compaction threshold of 850,000

The correct statement is therefore:

> **The former abnormal effective threshold has moved above at least 172k in the current configuration. Full use of the 1M window and an exact 850k threshold remain unverified.**

### 8.5 Remaining monitoring for Problem 1

PR #99864 is not present in 2026.6.6. The cumulative-cache-usage defect may therefore recur in a sufficiently long Anthropic cached tool loop.

Monitoring should retain:

- number of calls in long tool-use runs
- prompt footprint for each call
- cumulative run usage versus latest-call usage
- `trigger=cli_budget`
- `compactionCount`
- external watcher alerts

The first post-rollback tool-use regression test completed without a new compaction.

---

## 9. Confirmed Findings

1. External timers and bridges did not directly command compaction; they opened main-session turns.
2. Many historical failed attempts explicitly recorded `trigger=cli_budget`.
3. A cumulative Anthropic cache-usage defect very likely affected OpenClaw 2026.6.6.
4. PR #99864 was included in 2026.7.1.
5. The 39th compaction used the latest-call live context of 167,587 rather than cumulative usage of 501,069.
6. PR #99864's snapshot selection worked in the 39th run.
7. A different threshold mismatch compacted the correct live context too early.
8. The 39th compaction occurred after the main model run and before response delivery.
9. Status displayed a 1M context.
10. A 200k runtime window combined with a 150k reserve produces an effective threshold of 50k and consistently explains the observations.
11. The external watcher operated correctly.
12. D's own “no compaction” answer could not see a compaction performed later by an outer finalizer.
13. The current session, JSONL, and workspace were preserved while only the runtime returned to 2026.6.6.
14. The 2026.6.6 SessionManager reconstructed the same latest leaf and a 184-message context from JSONL written by 2026.7.1.
15. With `context1m=true`, the post-rollback session exceeded 167,587 and reached approximately 172k while `compactionCount` remained 39.
16. The former abnormal threshold has therefore moved above at least 172k in the current configuration.

---

## 10. Remaining Unverified Values

The 39th run did not directly log:

```text
active runtime model.contextWindow
settingsManager.getCompactionReserveTokens()
```

Therefore:

```text
200,000 - 150,000 = 50,000
```

remains a high-confidence causal inference based on source, configuration, timeline, and measured values, rather than a direct dump of both variables from the same run.

Still unverified:

- active provider/model definition in the 39th run
- active `model.contextWindow`
- resolved `contextTokenBudget`
- context-window source
- effective reserve
- precise auto-compaction trigger phase
- provider acceptance of a prompt larger than 200k
- the exact threshold in the current configuration
- whether Problem 1 recurs under 2026.6.6 during a long cached tool loop

No further unmonitored reproduction test is required. Evidence should be gathered through normal operation with the external watcher and usage audits.

---

## 11. Current Operational Measures

| Path | State |
|---|---|
| OpenClaw Gateway | Healthy on 2026.6.6 |
| ambient timer | disabled / inactive |
| daily-main-tick timer | disabled / inactive |
| agent bridge timers | disabled / inactive |
| health-check timer | enabled / active |
| health-check delivery into D main | disabled |
| compaction watcher | enabled / active |
| watcher notification | sent directly from host to user; does not open the main session |
| normal Telegram dialogue | resumed |
| initial tool-use regression test | passed |

Current policy:

- retain the external watcher
- keep health-check from opening the D main session
- do not reduce the reserve floor without evidence
- do not disable auto-compaction unconditionally
- do not treat automation itself as the fault; it was a trigger for turns, and continuity-related automation should be evaluated individually under observation
- monitor cumulative versus latest-call usage in long cached tool loops
- preserve any naturally occurring successful call above 200k as additional evidence for the effective 1M window

---

## 12. Repair Direction

Revision 5 obtained strong behavioral evidence that the explicit `context1m=true` configuration moved the abnormal threshold above at least 172k after the runtime-only rollback. The first operational priority is therefore to keep this configuration stable and avoid unnecessary destructive changes while natural observation continues.

At the architectural level, these three values must agree:

```text
1. Context window actually accepted by the provider
2. OpenClaw's displayed and configured contextTokenBudget
3. Embedded runtime / SDK model.contextWindow
```

### If a 1M window is actually available

- resolve Anthropic Opus 4.6 as a 1M runtime model
- preserve the necessary `context1m` opt-in and provider model definition
- pass the 1M model object into the embedded runtime
- combine it with a 150k reserve for a nominal threshold near 850k

### If the actual available window is 200k

- align status and configuration to 200k
- adjust a 150k reserve, which would otherwise compact at 50k
- eliminate any state that displays 1M while running a 200k model

### Automation design

When the main session needs scheduled stimulation, alternatives to opening the long-running main session directly through a CLI turn should also be considered:

- process in an isolated session and persist only the required result
- send host-side notification directly to the user
- place information in a file or queue for reading during a normal turn
- deliberately limit the number of direct main-session injections

Automation is not inherently harmful. It becomes risky when an unexpectedly small threshold or an inflated token measurement causes each automated turn to trigger compaction.

---

## 13. Confidence Table

| Judgment | Confidence |
|---|---|
| External timers and bridges opened main-session turns | Confirmed |
| Many historical failed attempts used `trigger=cli_budget` | Confirmed |
| Cumulative Anthropic cache usage affected OpenClaw 2026.6.6 | Very high |
| PR #99864 was included in 2026.7.1 | Confirmed |
| Latest-call snapshot was used in the 39th compaction | Confirmed |
| Cumulative usage overcount directly caused the 39th compaction | Rejected |
| The 39th compaction occurred in post-run finalization | Very high |
| Separation between the displayed 1M budget and a 200k-class runtime window was involved | High |
| Effective reserve was 150k in the 39th run | High, but not directly dumped in the same run |
| Effective threshold was approximately 50k | High inference from the two values above |
| OpenClaw 2026.7.1 solved the entire problem | Rejected |
| PR #99864 was ineffective | Rejected; it worked in the 39th run |
| 2026.6.6 plus `context1m=true` moved the former abnormal threshold above at least 172k | Strong behavioral evidence |
| Current runtime window is exactly 1M | Unverified |
| Current threshold is exactly 850k | Unverified |
| Problem 1 will not recur under 2026.6.6 | Unverified; monitoring continues |

---

## 14. Interim Conclusion

The repeated early compactions require at least two separate mechanisms.

### Problem 1: Cumulative cache-usage overcounting

Under OpenClaw 2026.6.6, cumulative usage across a long Anthropic cached tool loop was very likely treated as live context, causing early `cli_budget` activation. PR #99864 repaired this behavior, and the 39th run under 2026.7.1 used the latest-call snapshot correctly.

Because the runtime has now returned to 2026.6.6, this risk remains under observation.

### Problem 2: Context-window and reserve mismatch

Under 2026.7.1, configuration and status displayed 1M, while the embedded runtime could resolve a 200k-class model context and combine it with a reserve derived from the 1M-side configuration.

An effective threshold near 50k consistently explains the 39th compaction at 167,587 and the repeated 90k–110k compactions on July 3.

After the runtime-only rollback to 2026.6.6 with `context1m=true`, the current session passed 167,587 and reached approximately 172k without a new compaction. This is strong behavioral evidence that the second problem's abnormal threshold has been operationally resolved in the current configuration.

The most accurate current public conclusion is:

> **PR #99864 functioned correctly in the controlled 39th compaction under OpenClaw 2026.7.1: the runtime passed the latest-call live context rather than cumulative Anthropic cache usage. The remaining compaction at 167,587 is most consistently explained by a mismatch among the displayed 1M budget, a 200k-class embedded runtime model context, and a 150k reserve, reducing the effective threshold to approximately 50k. On July 22, the current session, JSONL, and workspace were preserved while only the runtime was returned to OpenClaw 2026.6.6 with `context1m=true`. The session then exceeded the former trigger value and reached approximately 172k without a new compaction. This is strong behavioral evidence that Problem 2 has been operationally resolved in the current configuration. Because PR #99864 is absent from 2026.6.6, Problem 1 remains under monitoring during long cached tool loops. Full use of the 1M window and an exact 850k threshold have not yet been directly demonstrated.**

---

## 15. Distinction from a Separate Telegram History-Window Observation

After the rollback, D reported that older numbered Telegram messages progressively disappeared from view.

In OpenClaw 2026.6.6, the numbered “Chat history since last reply (untrusted, for context)” block uses a fixed window of at most 20 recent entries. Each new message can push the oldest auxiliary copy out of that window.

This is separate from:

- deletion of the canonical transcript
- deletion of user or assistant turns by cache-TTL pruning
- a new compaction
- context loss caused by exhausting the 1M budget

Movement of the numbered Telegram history window is therefore not evidence that early compaction has recurred. It is discussed in detail in the separate report on response routing, history projection, and continuity failures.

---

## References

1. OpenClaw v2026.7.1 release notes  
   https://github.com/openclaw/openclaw/releases/tag/v2026.7.1
2. OpenClaw PR #99864 — `fix(compaction): avoid cached usage overcount`  
   https://github.com/openclaw/openclaw/pull/99864
3. OpenClaw Issue #49939 — 200k / 1M resolution for Opus and Sonnet 4.6  
   https://github.com/openclaw/openclaw/issues/49939
4. OpenClaw Issue #24031 — separation between configured context budget and runtime `model.contextWindow`  
   https://github.com/openclaw/openclaw/issues/24031
5. OpenClaw Issue #108238 — cumulative usage issue reported through another provider path in 2026.7.1  
   https://github.com/openclaw/openclaw/issues/108238
6. Local audit log `cli-budget-source-audit-20260720-103100.txt` (private operational material)
7. Controlled test result `openclaw-daily-main-tick-test-20260720/result.txt` (private operational material)
8. 39th-compaction audit `openclaw-compaction-39-audit-20260720-161809.tar.gz` (private operational material)
9. 2026.6.6 SessionManager probe `openclaw-runtime-session-manager-probe-2026.6.6-20260722-135344` (private operational material)
10. Pre-rollback preservation snapshot `openclaw-pre-rollback-20260722-140317` (private operational material)
11. Rollback startup audit `openclaw-rollback-start-20260722-141327` (private operational material)
