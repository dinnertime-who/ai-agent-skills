# State Machine

This reference is authoritative for `code-verifier-state.json`, located by searching the entire project directory. The default creation path is `<project-root>/docs/code-verifier/code-verifier-state.json`.

## States

| Status | Meaning |
| --- | --- |
| `idle` | No active criteria. Wait for a new verification request. State contains only `status`. |
| `preflight` | Check git diff and criteria validity before any implementation review. |
| `await-request` | Criteria accepted; wait for an explicit verification start request. |
| `processing-request` | Verify implementation and write the current feedback file. |
| `await-summary` | Score is 90+; wait for summary accept/refuse. |
| `process-summary` | Create or merge summary, delete only feedback files actually used, then return to `idle`. |

## State Shapes

`idle`:

```json
{ "status": "idle" }
```

`preflight`:

```json
{
  "status": "preflight",
  "criteriaPath": "path/to/example.md",
  "verifierDir": "docs/code-verifier"
}
```

`await-request` initial:

```json
{
  "status": "await-request",
  "criteriaPath": "path/to/example.md",
  "verifierDir": "docs/code-verifier",
  "feedbackCount": 0,
  "verificationType": "initial"
}
```

`await-request` after a failed pass:

```json
{
  "status": "await-request",
  "criteriaPath": "path/to/example.md",
  "verifierDir": "docs/code-verifier",
  "latestScore": 82,
  "feedbackCount": 1,
  "verificationType": "recheck"
}
```

`processing-request`:

```json
{
  "status": "processing-request",
  "criteriaPath": "path/to/example.md",
  "verifierDir": "docs/code-verifier",
  "feedbackCount": 2,
  "verificationType": "recheck",
  "feedbackPath": "docs/code-verifier/example-feedback-2.md"
}
```

`processing-request` after a prior failed pass may also carry the previous completed score:

```json
{
  "status": "processing-request",
  "criteriaPath": "path/to/example.md",
  "verifierDir": "docs/code-verifier",
  "latestScore": 82,
  "feedbackCount": 2,
  "verificationType": "recheck",
  "feedbackPath": "docs/code-verifier/example-feedback-2.md"
}
```

`await-summary`:

```json
{
  "status": "await-summary",
  "criteriaPath": "path/to/example.md",
  "verifierDir": "docs/code-verifier",
  "latestScore": 94,
  "feedbackCount": 2,
  "verificationType": "recheck"
}
```

`process-summary`:

```json
{
  "status": "process-summary",
  "criteriaPath": "path/to/example.md",
  "verifierDir": "docs/code-verifier",
  "latestScore": 94,
  "feedbackCount": 2,
  "verificationType": "recheck",
  "summaryPath": "docs/code-verifier/example-feedback-summary.md"
}
```

## Field Rules

- `criteriaPath` is required in every non-idle state. Must be a project-relative path (e.g. `docs/plans/example.md`). Never store as an absolute path; absolute paths break cross-machine sync.
- `verifierDir` is required in every non-idle state. It is always `docs/code-verifier` — the flat directory where all feedback, summary, and state files are stored.
- `feedbackCount` is required from `await-request` onward.
- `verificationType` is required from `await-request` onward.
- `latestScore` is the score of the latest completed feedback pass only. It never describes an in-progress or interrupted `processing-request`.
- `latestScore` is required after at least one review pass.
- `feedbackPath` exists only in `processing-request` and must point to a file inside `verifierDir`.
- `summaryPath` exists only in `process-summary` and must point to a file inside `verifierDir`.
- `feedbackCount` means the latest allocated feedback number for the active criteria.
- In `await-request`, `feedbackCount: 0` means initial verification; `feedbackCount >= 1` means recheck.
- `verificationType` records the current request type. When entering `processing-request`, first store the current count as `preIncrementFeedbackCount`, calculate `verificationType` from that value, then increment `feedbackCount`.

## Transitions

| From | Event / Condition | To | Required action |
| --- | --- | --- | --- |
| `idle` | user requests criteria verification | `preflight` | Store `criteriaPath`; check git diff before reading criteria. |
| `preflight` | git diff exists | `idle` | Ask user to clear/commit diff; do not read criteria. |
| `preflight` | criteria invalid | `idle` | Report missing plan requirements/verification criteria. |
| `preflight` | criteria valid | `await-request` | Summarize criteria only; wait for verification start. |
| `await-request` | user requests verification | `processing-request` | Store `preIncrementFeedbackCount`; set `verificationType` from it; increment `feedbackCount`; set `feedbackPath` in `verifierDir` from the incremented count. |
| `processing-request` | feedback written | score branch | Save `latestScore` before branching. |
| score branch | `latestScore < 90` | `await-request` | Keep same criteria; set `verificationType` to `recheck`. |
| score branch | `latestScore >= 90` | `await-summary` | Ask whether to summarize feedback. |
| `await-summary` | summary refused | `await-request` | Keep score; set `verificationType` to `recheck`. |
| `await-summary` | summary accepted | `process-summary` | Set `summaryPath`; run summary workflow. |
| `process-summary` | done | `idle` | Write only `{ "status": "idle" }`. |

## Resume Safety

Do not resume transient actions in place after an unexpected session stop.

| Interrupted status | Move to | Recovery action |
| --- | --- | --- |
| `preflight` | `idle` | Drop `criteriaPath`; ask user to request verification again. |
| `processing-request` | `await-request` | Decrement `feedbackCount` by 1, remove `feedbackPath`, recompute `verificationType` from the decremented count, keep `latestScore` only as the previous completed pass score if it existed before. |
| `process-summary` | `await-summary` | Remove `summaryPath`; ask user to accept summary again. |

Safe points are only `idle`, `await-request`, and `await-summary`.

When recovering from `processing-request`, tell the user that the interrupted pass was discarded and any preserved `latestScore` is from the previous completed pass, not from the interrupted pass. Do not imply that a new verification score exists until feedback has been written and `latestScore` has been saved again.
