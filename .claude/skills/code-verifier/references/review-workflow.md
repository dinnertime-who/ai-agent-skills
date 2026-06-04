# Review And Summary Workflow

## Plan Validity

A criteria file is valid only when it exists and states:

- implementation requirements,
- verification criteria,
- concrete conditions that can become a checklist.

Do not inspect implementation code during preflight.

## Review Procedure

When entering `processing-request`:

1. Read the criteria file.
2. Read prior feedback files only when `verificationType` is `recheck`.
3. Prior feedback files are numbered files for the same criteria with suffixes lower than the current `feedbackCount`; exclude the current `feedbackPath` and any summary.
4. Run `git diff --stat`, `git diff --name-only`, and relevant `git diff` inspection.
5. Inspect changed files and related code paths: callers, stores, composables, schemas, tests, docs, and integration boundaries when relevant.
6. Verify each criterion and each prior finding against source code.
7. Assign a score out of 100.
8. Write feedback using `assets/feedback-template.md`.
9. Save `latestScore` immediately after feedback is written, then transition by score.

Use `rg` or `rg --files` for searches when available.

## Scoring

- `90-100`: criteria met; no material bug found.
- `80-89`: mostly correct with at least one meaningful issue or gap.
- `60-79`: partial implementation with important missing coverage.
- `< 60`: major failure, broken behavior, or unsafe to validate.

Below 90, keep the same criteria locked until a later verification reaches 90 or higher.

## Feedback File Policy

Feedback file location and name:

```text
<criteria-directory>/<criteria-basename>-feedback-<feedbackCount>.md
```

Store every feedback file in the same directory as the criteria file.

If the target feedback file already exists:

- Compare it with `assets/feedback-template.md`.
- If required sections are missing or malformed, overwrite it.
- If structurally valid, merge existing and new content.

During merge, new generated content wins for:

- `Score`
- `Verdict`
- `Findings`
- `Criteria Check`
- `Prior Feedback Status`

Preserve valid human-written notes only from `Notes`. Human edits in generated sections may be replaced by the new verification result.

For initial verification, `Prior Feedback Status` must be `None`.

For rechecks, mark prior findings as `Fixed`, `Still failing`, `Partially fixed`, or `Superseded`, with short evidence.

## Summary Workflow

Before summarizing, compare `feedbackCount` with expected suffixes `1..feedbackCount`.

Available feedback means the file exists and matches the feedback template structure. Missing feedback files are ignored; do not restore from git and do not roll back current changes.

If no available feedback exists:

- Do not create or edit a summary file.
- Do not delete any existing summary file.
- Set state to `{ "status": "idle" }`.

If at least one feedback file is available:

1. Summarize only available feedback files.
2. Record missing ignored feedback files in `Notes`.
3. Write or merge `<criteria-directory>/<criteria-basename>-feedback-summary.md` using `assets/summary-template.md`.
4. Fix the list of feedback files actually used by the summary.
5. Delete only those used feedback files.
6. Set state to `{ "status": "idle" }`.

## Summary File Policy

If the target summary file already exists:

- Compare it with `assets/summary-template.md`.
- If required sections are missing or malformed, overwrite it.
- If structurally valid, merge existing and new content.

During merge, new generated content wins for:

- `Final Outcome`
- `Feedback History`
- `Resolved Issues`
- `Residual Risks`

Preserve valid human-written notes only from `Notes`.
