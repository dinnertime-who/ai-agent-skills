---
name: code-verifier
description: Verify implementation quality against a user-specified markdown criteria file by enforcing a persistent verifier state machine, inspecting git diff and related code only after preflight, writing numbered scored feedback files, requiring same-criteria rechecks below 90/100, and optionally summarizing feedback after a passing score. Use when asked to verify, reverify, continue, score, write feedback for, or summarize implementation work against criteria markdown.
---

# Code Verifier

Act as a verifier, not an implementer. Do not modify product code unless the user explicitly changes the task from verification to implementation.

## Required State Gate

At the start of every verifier turn:

1. Search the entire project directory for `code-verifier-state.json`. Use the first one found if multiple exist (log a warning if more than one is found).
2. If no `code-verifier-state.json` exists anywhere in the project, treat state as `{ "status": "idle" }`.
3. Normalize interrupted transient states using `references/state-machine.md`.
4. Reject any action that is not allowed by the active state.
5. If `status` is not `idle`, reject other criteria paths; the stored `criteriaPath` is locked.

Only `idle` may accept a new criteria verification request. `idle` state must contain only:

```json
{ "status": "idle" }
```

## What To Load

Load only what is needed:

- Always read `references/state-machine.md` when state is not `idle`, when updating state, or when handling verification/summary.
- Read `references/review-workflow.md` when entering `processing-request` or `process-summary`.
- Use `assets/feedback-template.md` for feedback files.
- Use `assets/summary-template.md` for summary files.

## High-Level Flow

State file path: `<project-root>/docs/code-verifier/code-verifier-state.json`. This is the default location when creating a new session; create the directory if needed. The file may exist elsewhere if the user placed it there previously.

Supported statuses and resume rules are defined in `references/state-machine.md`. Never continue a transient action from the middle.

## Entry Rules

When `idle` and the user provides criteria or asks for verification:

1. Set state to `preflight` with `criteriaPath` stored as a project-relative path (never absolute).
2. Run `git diff --stat` and `git diff --name-only`.
3. If any diff exists, ask the user to clear/commit it, set `idle`, and do not read criteria.
4. If clean, read only criteria and validate implementation requirements plus verification criteria.
5. If the plan is invalid, report why and set state to `idle`.
6. If valid, summarize criteria only, set `await-request` with `feedbackCount: 0` and `verificationType: "initial"`, then wait.

When `await-request` and the user explicitly requests verification:

1. Store the current `feedbackCount` as `preIncrementFeedbackCount`.
2. Set `verificationType` from `preIncrementFeedbackCount`: `0` means `initial`; `1+` means `recheck`.
3. Increment `feedbackCount` by 1.
4. Set `feedbackPath` under the verifier directory: `<project-root>/docs/code-verifier/<criteria-basename>-feedback-<feedbackCount>.md`, using the incremented count.
5. Set state to `processing-request`.
6. Run `references/review-workflow.md` and write/merge feedback.
7. Save `latestScore` first, then branch:
   - `< 90`: set `verificationType` to `recheck`, return to `await-request`.
   - `>= 90`: move to `await-summary`; ask whether to summarize.

When `await-summary`:

- If accepted, keep `latestScore`, `feedbackCount`, and `verificationType`; set `summaryPath`, move to `process-summary`, and run summary.
- If the user refuses summary, keep `latestScore`, set `verificationType` to `recheck`, return to `await-request`, and allow only the same criteria to be reverified.

When `process-summary` completes, always set state to `{ "status": "idle" }`.

## Review Stance

Use a code-review stance. Findings come first and must be source-backed. Verify claims against changed code and related paths, not only criteria or prior feedback.

Write user-facing responses and feedback in the user's language unless asked otherwise. Score with the rubric in `references/review-workflow.md`; below 90 keeps the same criteria locked.

## File Naming

All verifier artifacts live under `<project-root>/docs/code-verifier/`.

For `example.md`, write:

- State: `docs/code-verifier/code-verifier-state.json` (default; actual location found by project-wide search)
- Feedback: `docs/code-verifier/example-feedback-<feedbackCount>.md`
- Summary: `docs/code-verifier/example-feedback-summary.md`

The criteria file itself stays in its original location and is never moved or copied.

If a target file already exists, compare it with the relevant template. Overwrite it only when structurally incomplete. If structurally valid, merge old and new content using the section rules in `references/review-workflow.md`.
