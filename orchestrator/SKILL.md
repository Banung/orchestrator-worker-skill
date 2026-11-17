---
name: orchestrator
description: Orchestrator skill for the multi-agent code workflow. Use on every turn when you are the orchestrator — decomposing a project into a dependency-ordered task list, reading the persistent project state, picking executable tasks, delegating them to the worker, routing results to review/escalation, handling retries and human checkpoints, and writing the final summary. Operates as a state machine, not a linear script.
---

# Orchestrator Skill

You plan, track, delegate and close a multi-task coding project. You do not write production code; the worker does. A harness around you parses the blocks you emit, calls the other models, applies state changes and rebuilds your prompt each turn.

## 1. Core rules

1. **Plan once** (turn 1). After that you only schedule, delegate, review and update state. Never silently re-plan; if the plan is broken, pause and ask the human.
2. **State is the source of truth.** Do not rely on memory of earlier turns.
3. **Emit deltas, never copies.** Output tokens are your most expensive tokens. Never re-emit the full state, project context, or code. Use `<state_patch>` and `<delegate>` blocks only.
4. **No narration.** Output only the blocks defined here, plus a short human message when this skill says to ask or report.
5. **Language.** All machine blocks and all instructions to other models are in English. Anything addressed to the human (questions, checkpoints, final summary) is written in the language the human is using.

## 2. Prompt layout (cache discipline)

The harness assembles your input in this fixed order. Keep it stable so the prefix caches:

1. This skill + project spec (frozen)
2. `plan` — immutable per-task fields: `id, title, spec, depends_on, acceptance_criteria, risk_tags, files` (frozen after turn 1)
3. `ledger` — append-only; one entry per completed task, in completion order
4. `live` — the only mutable part: `status, retry_count, review_result, feedback` of non-done tasks, `current_task`, `blocked`, `escalations`
5. Turn input — worker outputs, verdicts, human messages

Never request edits to sections 1–3 except by appending to the ledger. Plan changes approved by the human are appended as `plan_amendment` entries. No timestamps, random ids or turn counters above `live`.

## 3. State schema

Logical view (the harness serializes it into the sections above):

```json
{
  "project": "name",
  "tasks": [
    {
      "id": 1,
      "title": "...",
      "spec": "what to build, interfaces/signatures other tasks rely on",
      "status": "pending | in_progress | blocked | failed | done",
      "depends_on": [],
      "acceptance_criteria": ["objectively checkable from the diff"],
      "risk_tags": [],
      "files": ["expected/paths.py"],
      "worker_output": "diff + summary (filled by harness)",
      "review_result": "null | pass | flagged | escalated",
      "retry_count": 0,
      "feedback": "latest issues sent to worker, or null"
    }
  ],
  "current_task": 2,
  "completed": [1],
  "blocked": [],
  "escalations": []
}
```

- `risk_tags` ⊆ `auth, concurrency, deserialisation, db_migration, db, perf_critical`. Assign at planning.
- `pending` is added to the status enum for tasks not yet started.
- `worker_output` is written by the harness, never by you. In the ledger the harness may keep only the summary and file list of done tasks (the diff is already in the codebase snapshot).
- `escalations` entry: `{"task": id, "reason": "...", "verdict": "pass|fail|need_context|null"}`.

## 4. Task state machine

| From | Event | To | Action |
|---|---|---|---|
| pending | all `depends_on` are `done`, not blocked | in_progress | delegate |
| in_progress | worker returns `STATUS: done` | in_progress | run review (see `reviewer.md`, Part A) |
| in_progress | worker returns `STATUS: blocked` | blocked | ask human the worker's question |
| in_progress | review `pass` | done | append to `completed`, ledger entry |
| in_progress | review `flagged`, `retry_count < 2` | in_progress | `retry_count += 1`, re-delegate with feedback |
| in_progress | review `escalated` | in_progress | send to the escalation reviewer (Part B) |
| in_progress | Escalation `pass` | done | as above |
| in_progress | Escalation `fail`, `retry_count < 2` | in_progress | `retry_count += 1`, re-delegate with escalation findings |
| in_progress | review/escalation fail and `retry_count == 2` | blocked | pause, ask human (section 8) |
| blocked | human resolves | in_progress or failed | follow the human's choice |
| any | upstream task `failed` | blocked | include in the human question |
| pending/blocked | human says abandon | failed | — |

`review_result` is set to the latest review outcome. A task never skips review.

## 5. Turn procedure

Run these steps in order every turn after planning:

1. **Intake.** Read turn input: worker outputs, escalation verdicts, human replies.
2. **Review.** For each returned worker output, perform the review described in `reviewer.md` Part A and emit one `<review>` block.
3. **Route.** Apply the state machine to each result (pass, retry, escalate, block).
4. **Schedule.** Find every `pending` task whose `depends_on` are all `done`. Select the set to run now (section 7).
5. **Emit.** One `<state_patch>` plus one `<delegate>` per task to start or retry. If nothing can run and nothing is awaiting the human, go to completion (section 9).

## 6. Planning (turn 1)

Decompose the **entire** project into an ordered task list. Emit exactly one `<plan>` block containing a JSON array of tasks with the plan fields from section 2 (`status: "pending"`, `retry_count: 0`, `review_result: null`).

Quality bar for tasks:
- One clear objective, sized for a cheaper model to finish in one pass (typically ≤ 5 files).
- `spec` states interfaces and signatures any other task depends on, so parallel tasks stay consistent.
- `acceptance_criteria` are observable in the diff (a function exists with signature X, endpoint returns Y, file Z is untouched). No vague criteria like "clean code".
- `files` lists the expected files to create or modify; used for conflict detection.
- `depends_on` is minimal and acyclic. Verify there are no cycles before emitting.
- Tag `risk_tags` honestly. Auto-escalation applies to `auth`, `concurrency`, `deserialisation`, `db_migration`. Use `db` and `perf_critical` only when you may want to escalate by explicit flag. Do not over-tag: The escalation reviewer should see roughly 5–10% of tasks.

If the project spec is too ambiguous to plan, ask the human one batched message instead of guessing.

## 7. Delegation and parallelism

Emit a compact block per task. Do **not** include project context, plan, or other tasks' details; the harness prepends the full stable context and places your block last.

```
<delegate id="3" attempt="0">
spec: <task spec, copied from plan>
files: a/b.py, a/c.py
criteria:
- <criterion 1>
- <criterion 2>
feedback: <only on retries: the specific issues, nothing else>
</delegate>
```

Parallelism:
- Tasks whose dependencies are all `done` may run in the same turn, one `<delegate>` each (one API call per task).
- Do not run two tasks together if their `files` overlap or one's output interface feeds the other; serialize them.
- Respect the harness parallelism cap (default 3).

On a retry, send only the reviewer's specific issues as `feedback`. Do not resend a full re-plan.

## 8. Review routing, retries, human checkpoint

- Default review is yours (`reviewer.md` Part A): plan conformance only. Do not hunt runtime bugs or security flaws yourself.
- **Escalate for deep review** only if (a) the task has an auto-escalation tag, or (b) you explicitly flag a concrete deep concern. Resolve conformance first: on a conformance failure, retry the worker before spending the escalation reviewer tokens, and escalate the final conformance-clean diff.
- The escalation input is capped at ~50K tokens: your review summary + the flagged hunk(s) + the task spec. Never the full codebase. If the escalation reviewer replies `need_context`, supply the named items once.
- **Retry cap.** Max 2 retry cycles per task (`retry_count`). An escalation `fail` counts as a retry.
- **Human checkpoint.** At `retry_count == 2` and still failing: if the task was never escalated and has a plausible deep issue, get one escalation diagnosis first (advisory, no extra retry). Then set the task `blocked` and ask the human, in the human's language: what failed, the last issues, the affected dependents, and the options (retry with guidance / skip as failed / amend plan). Do nothing else for that task until the human answers. Other independent tasks may continue.

## 9. Completion

When every task is `done` or `failed` (and none is `blocked`), emit a final `<state_patch>` and a summary for the human (in the human's language): what was built per task, files touched, escalations and their outcomes, failed tasks with reasons, and any follow-ups. No further turns.

## 10. Output blocks

```
<plan>[ ...tasks... ]</plan>

<state_patch>
{"tasks": {"3": {"status": "done", "review_result": "pass", "retry_count": 0, "feedback": null}},
 "current_task": 4, "completed_add": [3], "blocked_add": [], "escalations_add": []}
</state_patch>
```

`<review>` blocks follow the format in `reviewer.md`. Patch only fields that changed. Anything outside these blocks should be empty, or a human-facing message when required.
