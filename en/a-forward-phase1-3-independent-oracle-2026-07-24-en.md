# A-forward Phase 1.3: Opening the Independent Oracle

## A validation procedure in which a prior prediction is revealed only after the implementation has been frozen

**July 24, 2026**

Phase 1.3 began the independent reconciliation of the **A-forward Context Projection Guard**, which removes only non-terminal assistant text when an AI agent’s response history is projected back into the next model call, while preserving tool calls and the terminal final answer.

Before Q—the implementer—created the implementation, tests, and Implementation Record, VecTA independently produced the expected outputs as an Oracle and sealed them.

Without opening the Oracle, Q implemented the pure projector and offline wrapper, completed 48 focused tests, type-checked the source and tests, and completed linting and formatting. The implementation was then frozen as Phase 1.2. Only after that freeze was the Oracle opened.

The preliminary comparison immediately after opening found no semantic disagreement on the following central principles:

- For non-terminal assistant passes, remove only text blocks.
- Preserve tool calls, tool results, ordering, and pairing.
- Leave terminal final answers unchanged.
- Use tool-call structure—not a phase label such as `commentary`—as the primary classification signal.
- Do not apply the policy to history before the cutoff.
- Preserve historical runs that ended without a terminal final answer, following the conservative fail-open rule.

This did not yet mean that final validation was complete. The next step was a formal comparison against the actual A1/A2 fixtures, including projected messages, the Decision Log, tool pairing, and the identity of the terminal final answer.

## Update — July 26, 2026

DenneTA completed the Phase 1.5 review.

D approved the conservative fail-open boundary, selected:

```text
messagingTextPolicy = strip
```

and accepted the limited A-only scope without conditions.

Marina has authorized only the preparation of the runtime-integration and activation plans. Implementation, deployment, activation, and live testing remain unapproved.

---

[← Back to Top](/en/)
