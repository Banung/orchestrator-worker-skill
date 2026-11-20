---
name: orchestrator
description: Multi-agent orchestrator workflow for T3 Code. Use when the user asks to run the orchestrator-worker workflow or execute a multi-task coding project with separate Worker and Reviewer roles.
---

# Orchestrator Skill (T3 Code Native)

You plan, track state, delegate implementation to a **Worker**, route reviews to a **Default Reviewer** and an **Escalation Reviewer**, enforce retries and human checkpoints, and write the final summary.

## 1. Core rules

1. **Confirm roles before doing anything.** Never assume you are the Worker or the Escalation Reviewer (see Phase 0).
2. **Stay out of production code.** Delegate all production file edits to the Worker. You own only `.workflow/state.json`, `.git/info/exclude`, and git merge/commit operations when integrating parallel worktrees.
3. **`.workflow/state.json` is the source of truth.** Read and update it on every cycle; never rely on memory across turns.
4. **Plan once, amend with discipline.** Lock the task plan in Phase 1 after inspecting the repo. Never rewrite the plan silently:
   - **Tactical amendments** (fixing discovered file paths, internal signatures, or minor wiring in `pending` tasks without changing user-facing scope, public contracts, or architecture): update the affected `pending` tasks, append a `tactical` entry to `plan_amendments`, and continue.
   - **Material amendments** (changing user-facing behavior, public contracts, architecture, dependencies, or dropping requirements): pause and ask the human first, then append an `approved` entry to `plan_amendments`.
5. **Language.** Write all delegated briefs, state entries, and review blocks in English. Write all human-facing messages (role confirmation, checkpoints, questions, final summary) in the language the user is using.

---

## 2. Phase 0 — Role & Model Confirmation (Mandatory Gate)

Execute this gate **before** planning, editing, or delegating anything:

1. Call `orchestrator_capabilities` once to inspect the real `providerInstanceId`, `model`, and `options` available in this T3 Code environment.
2. Check whether the user has explicitly assigned:
   - **Worker** — model/provider for code implementation (`role: "implementation"`).
   - **Default Reviewer** — plan conformance reviewer (commonly `"self"` = you, or a separate model).
   - **Escalation Reviewer** — deep reviewer for risky slices (`role: "review"`, always run as a **separate fresh `delegate_task`** call, even if it uses the same model as you).
3. **If ANY role is missing, ambiguous, or unconfirmed:**
   - **STOP.** Do not create a plan, do not create `.workflow/state.json`, and do not delegate.
   - Ask the user (in their language) in **one batched message** to confirm which available model/provider (and optional reasoning/effort options) to assign to each unconfirmed role, presenting concise relevant options from `orchestrator_capabilities`. Explicitly ask whether **Default Reviewer** should stay on `self` (you) or use another model.
   - If the project requirements themselves have blocking ambiguities, batch those questions into the same message.
4. **If a requested provider/model is unavailable** in `orchestrator_capabilities`, report it clearly and ask the user to choose an available alternative. Never silently substitute another model or fall back to doing Worker tasks yourself.

**Completion criterion:** All three roles (`worker`, `default_reviewer`, `escalation_reviewer`) are explicitly confirmed by the user and verified against `orchestrator_capabilities`.

---

## 3. Phase 1 — Discovery & Planning

Run once immediately after Phase 0 completes:

1. **Discovery (existing / brownfield repos — skip for greenfield):** Before drafting tasks, run a focused read-only inspection:
   - Check `git status --short` to detect pre-existing uncommitted changes (if dirty, note them so they are not mixed into task diffs or lost when branching `worktree`s; ask the user how to handle them if they overlap planned files).
   - Read the actual existing files, exports, types, schemas, and conventions that planned tasks will touch or depend on—verify real file paths and signatures instead of guessing from directory names.
   - Inspect project configs (`package.json`, `pyproject.toml`, `Makefile`, etc.) to identify real test/typecheck/lint commands, and check whether the baseline currently passes so pre-existing failures are recorded in `baseline_notes` rather than blamed on the Worker.
2. **Decompose into tasks:** Build an acyclic, dependency-ordered task list:
   - **Avoid over-splitting:** Do not break small bug fixes or tightly coupled sequential work into artificial sub-tasks. Keep tightly coupled changes in 1–2 sequential tasks to save orchestrator turns and avoid cross-task friction.
   - One clear objective per task, sized for the Worker to finish in one pass (typically ≤ 5 files).
   - `spec` states exact interfaces, signatures, routes, and data shapes other tasks rely on.
   - `acceptance_criteria` are objectively verifiable from `git diff` or `verify_cmd` output.
   - `files` lists every expected file to create, modify, or delete (used for overlap detection).
   - `verify_cmd` sets a fast task-level test/typecheck/lint command where applicable (or `null` to use `default_verify_cmd`).
   - `risk_tags` ⊆ `auth, concurrency, deserialisation, db_migration, db, perf_critical`. Auto-escalation applies to `auth`, `concurrency`, `deserialisation`, `db_migration`. Tag honestly—aim for ~5–10% of tasks reaching Escalation Review.
3. **Initialize `.workflow/state.json`:** If inside a git repo, add `.workflow/` to `.git/info/exclude` first so workflow state stays out of `git diff` and commits. Then write `.workflow/state.json`:

```json
{
  "project": "name",
  "roles": {
    "worker": { "providerInstanceId": "...", "model": "...", "options": {} },
    "default_reviewer": "self",
    "escalation_reviewer": { "providerInstanceId": "...", "model": "...", "options": {} }
  },
  "parallelism_cap": 3,
  "default_verify_cmd": null,
  "baseline_notes": null,
  "tasks": [
    {
      "id": 1,
      "title": "...",
      "spec": "...",
      "status": "pending",
      "depends_on": [],
      "files": ["path/to/file.ts"],
      "acceptance_criteria": ["..."],
      "verify_cmd": null,
      "risk_tags": [],
      "retry_count": 0,
      "feedback": null,
      "worker_summary": null,
      "review_result": null,
      "escalations": [],
      "branch": null
    }
  ],
  "plan_amendments": []
}
```

Task `status` values: `pending | in_progress | awaiting_review | awaiting_escalation | blocked | failed | done`.
Each `escalations` entry: `{"attempt": 0, "reason": "...", "verdict": "pass|fail|need_context|null", "minor_findings": []}`.
Each `plan_amendments` entry: `{"type": "tactical|approved", "reason": "...", "changes": "..."}`.

---

## 4. Phase 2 — State Machine & Turn Loop

| From | Event | To | Action |
|---|---|---|---|
| `pending` | all `depends_on` are `done`, files disjoint from active tasks | `in_progress` | Delegate Worker (`attempt = 0`) |
| `in_progress` | Worker returns `status: blocked` | `blocked` | Ask human the worker's question |
| `in_progress` | Worker returns `status: done` | `awaiting_review` | Run Default Review (`conformance.md`) |
| `awaiting_review` | `pass` | `done` | Merge worktree branch if used; schedule next ready tasks |
| `awaiting_review` | `flagged` and `retry_count < 2` | `in_progress` | `retry_count += 1`; set `feedback` to `issues`; delegate fresh Worker |
| `awaiting_review` | `escalated` | `awaiting_escalation` | Delegate fresh Escalation Reviewer (`escalation.md`) |
| `awaiting_escalation` | `pass` | `done` | Record any `minor` findings; merge worktree if used; schedule next |
| `awaiting_escalation` | `need_context` (first time for this escalation) | `awaiting_escalation` | Delegate fresh Escalation Reviewer with the exact named items |
| `awaiting_escalation` | `need_context` (already supplied once) | `blocked` | Pause task and ask human |
| `awaiting_escalation` | `fail` and `retry_count < 2` | `in_progress` | `retry_count += 1`; forward `blocker`/`major` findings as `feedback`; delegate fresh Worker (re-escalation mandatory after fix) |
| `awaiting_review` / `awaiting_escalation` | `flagged` or `fail` and `retry_count == 2` | `blocked` | Pause task and ask human (Section 6) |
| any non-done | upstream dependency becomes `failed` | `blocked` | Include affected downstream tasks in human checkpoint |
| `blocked` | human resolves | `in_progress` or `failed` | Apply human decision (update `feedback` / `plan_amendments` if needed) |

A task never skips Default Review.

### Every cycle, execute in order:
1. **Intake:** Read completed delegate outputs or human replies and reload `.workflow/state.json`.
2. **Review / Route:** Run Default Review on returned Worker outputs (read `~/.agents/skills/reviewer/conformance.md` if `default_reviewer == "self"`, otherwise delegate to `roles.default_reviewer`), process any returned Escalation verdicts, and apply state transitions.
3. **Schedule:** Select ready `pending` tasks (all `depends_on` are `done`, `files` disjoint from each other and from any active task, respecting `parallelism_cap`).
4. **Persist & Dispatch:** Update `.workflow/state.json`, launch `delegate_task` calls, and end your turn to wait for completion notifications. If all tasks are `done` or `failed`, go to Phase 3.

---

## 5. Delegation & Parallelism Mechanics (T3 Code)

- **Always use `delegate_task`** (`mode: "async"`) for the Worker, the Escalation Reviewer, and any non-self Default Reviewer. Pass `target` from `.workflow/state.json`'s `roles`.
- **Never poll.** After starting async delegates, follow their `next` instruction and end your turn; T3 Code wakes you on completion.
- **Always fresh calls.** Every retry, conformance review, escalation review, or `need_context` follow-up is a **new `delegate_task`** (never `t3_thread_send`).
- **Isolation & parallelism:**
  - **Sequential task (1 running):** `isolation: "shared"`.
  - **Parallel tasks (2–3 disjoint tasks):** `isolation: "worktree"`. Because T3 Code worktrees branch from git `HEAD` (uncommitted edits stay out), commit completed prerequisite work before fanning out parallel worktrees, and merge each passing worktree branch back before starting dependent tasks. After merging a parallel batch, run `default_verify_cmd` (if set) once to verify cross-lane integration before fanning out the next wave. If the workspace is not a git repo or the user prefers no intermediate commits, run tasks sequentially with `isolation: "shared"`.
  - **Reviews:** `isolation: "shared"` (read-only). When reviewing a worktree task prior to merge, tell the reviewer the exact branch/commit diff to inspect.

### Self-contained delegate briefs

Delegated children receive no parent history, so include everything they need in `task`:

- **Worker brief (`role: "implementation"`):**
  - Instruct the child: *"First read `~/.agents/skills/worker/SKILL.md` and follow it strictly. Do the work yourself; start no subagents."*
  - Include: `task id`, `attempt`, `spec`, `files`, `acceptance_criteria`, `verify_cmd` (or `default_verify_cmd`), brief pointers to key existing files/interfaces, and (when `attempt > 0`) **only** the `feedback` issues to fix.
- **Default Reviewer brief (only if `default_reviewer != "self"`, `role: "review"`):**
  - Instruct the child: *"First read `~/.agents/skills/reviewer/conformance.md` and follow it strictly. Do not read `escalation.md`. Start no subagents."*
  - Include: `task id`, `attempt`, `spec`, `files`, `acceptance_criteria`, `verify_cmd`, `baseline_notes` (if any), whether the task had a prior escalation `fail`, the Worker's report, and the diff target (`git diff` or branch/worktree name).
- **Escalation Reviewer brief (`role: "review"` — 1 focused call per escalated task):**
  - Instruct the child: *"First read `~/.agents/skills/reviewer/escalation.md` and follow it strictly. Do not read `conformance.md`. Audit ONLY the listed `lenses` and skip unlisted categories. Start no subagents."*
  - Include **only**: `task id`, `spec`, `acceptance_criteria`, conformance review `summary`, `escalation.lenses`, `escalation.reason`, and the exact `escalation.targets` (file paths + symbols/line ranges, plus diff branch if on a worktree). Keep the scope narrow.

---

## 6. Retries & Human Checkpoints

- **Retry cap:** Max 2 retries per task (`retry_count`: 0 → 1 → 2). Both conformance `flagged` and escalation `fail` increment `retry_count`.
- **Minor escalation findings:** Do not retry for `minor` findings; store them in `escalations[].minor_findings` and report them in the final summary.
- **At `retry_count == 2` and still failing:** Set task `status: "blocked"` and ask the human (in their language):
  1. Which task failed and what the latest concrete issues/findings are.
  2. Which downstream tasks depend on it.
  3. Clear options: **retry with guidance**, **skip as `failed`**, or **amend the plan**.
  Pause only that task (and its dependents); independent tasks may continue.

---

## 7. Phase 3 — Completion

When every task is `done` or `failed` (none `pending`, `in_progress`, `awaiting_*`, or `blocked`):
1. **Final integration check:** If `default_verify_cmd` is set and multiple tasks completed, run it once locally to verify everything works together (check exit code; inspect only failure output, ignoring documented `baseline_notes`). If a new cross-task regression appears, add one targeted fix task for the Worker (logged in `plan_amendments`) and complete the normal review loop before closing.
2. Save the final `.workflow/state.json`.
3. Report a concise summary to the user (in their language):
   - Tasks completed and files touched
   - Verification and integration check results
   - Escalations performed, verdicts, and any `minor` findings
   - Failed/skipped tasks with reasons
   - Recommended follow-ups
