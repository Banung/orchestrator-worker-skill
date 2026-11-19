# Multi-Agent Code Workflow Skills (T3 Code Native)

Three role-based agent skills for running a structured multi-model coding workflow natively inside **T3 Code**: one model plans, tracks state, and verifies conformance; a cheaper/faster model implements tasks in the workspace; and a fresh, isolated deep-review call audits only the risky slices.

## What's included

- [orchestrator/SKILL.md](orchestrator/SKILL.md): plans the project once (after discovery on existing repos), confirms model assignments via `orchestrator_capabilities`, tracks state in `.workflow/state.json`, delegates tasks via `delegate_task`, routes reviews and retries, and writes the final summary. Does not write production code.
- [worker/SKILL.md](worker/SKILL.md): stateless implementation worker. Edits files directly in the workspace using native tools, runs the task verification command, and returns a compact YAML report.
- [reviewer/SKILL.md](reviewer/SKILL.md): thin router splitting the two review stages into isolated files so neither leaks into the other's context:
  - [reviewer/conformance.md](reviewer/conformance.md): **Default Reviewer** — plan conformance and verification evidence only.
  - [reviewer/escalation.md](reviewer/escalation.md): **Escalation Reviewer** — independent deep audit (security, concurrency, deserialisation, database migrations, flagged hot paths) in a fresh delegated call.

## Roles & model assignment

The skills are model-agnostic. Example assignment:

| Role | Typical Model | Skill |
|---|---|---|
| Orchestrator + Default Reviewer | Strong planning model (e.g., Gemini 4 Argon) | `orchestrator/SKILL.md`, `reviewer/conformance.md` |
| Worker | Fast / low-cost coding model (e.g., DeepSeek V4.1 Flash) | `worker/SKILL.md` |
| Escalation Reviewer | Strong reasoning model (fresh isolated `delegate_task` call) | `reviewer/escalation.md` |

### Mandatory role confirmation (Phase 0)

If you invoke the orchestrator without specifying which models should play **Worker**, **Default Reviewer**, and **Escalation Reviewer**, the orchestrator calls `orchestrator_capabilities` to list your real available providers/models and **stops to confirm with you** before planning or delegating anything. It will never silently assume it is the Worker or substitute unavailable models.

## How it works

```
Orchestrator (Phase 0)  ->  confirm roles via orchestrator_capabilities
Orchestrator (Phase 1)  ->  discover repo -> plan -> write .workflow/state.json
Orchestrator (Phase 2)  ->  delegate_task (Worker) -> Default Review (conformance.md)
                              ├── pass      -> mark done -> schedule next
                              ├── flagged   -> retry Worker with issues (max 2 retries)
                              └── escalated -> fresh delegate_task (escalation.md)
                                                 ├── pass         -> mark done
                                                 ├── fail         -> retry Worker (re-escalation mandatory)
                                                 └── need_context -> supply named items once
```

1. **Role confirmation (Phase 0).** Queries `orchestrator_capabilities` and confirms `worker`, `default_reviewer` (`self` or separate model), and `escalation_reviewer`.
2. **Discovery & planning (Phase 1, once).** On existing (brownfield) repos, checks `git status`, reads real target files/interfaces/conventions, and records `baseline_notes` for pre-existing test/build state before decomposing the project into an acyclic task graph with expected `files`, `acceptance_criteria`, `verify_cmd`, and `risk_tags`. Initializes `.workflow/state.json` (excluded from git via `.git/info/exclude`). Tactical adjustments to downstream `pending` tasks discovered mid-flight are logged in `plan_amendments`; material scope/architecture changes still pause for human approval.
3. **Delegation & parallelism (Phase 2).** Dispatches `delegate_task` (`mode: "async"`) for ready tasks:
   - Single task: `isolation: "shared"`.
   - Parallel independent tasks (disjoint `files`, up to `parallelism_cap = 3`): `isolation: "worktree"` (committing completed prerequisites first and merging passing worktrees back) or sequential `shared` if no intermediate commits are desired.
4. **Worker.** Implements only its assigned task directly with native tools, runs `verify_cmd`, and returns a compact `worker_report`.
5. **Default Review (`conformance.md`).** Checks actual `git diff` and verification output against the plan and acceptance criteria. Fixes conformance failures before escalating.
6. **Escalation Review (`escalation.md`).** Runs in a **fresh `delegate_task`** only when conformance is clean and the task touches `auth`, `concurrency`, `deserialisation`, `db_migration`, previously failed escalation, or triggers a concrete deep concern (~5–10% of tasks).
7. **Retries & human checkpoints.** Max 2 retries per task (`flagged` or escalation `fail`). At `retry_count == 2` and still failing, the task pauses as `blocked` and asks you whether to retry with guidance, skip as `failed`, or amend the plan.
8. **Completion (Phase 3).** Once all tasks are `done` or `failed`, writes a final summary in your language covering deliverables, verification results, escalations, minor findings, and any follow-ups.

## Project state (`.workflow/state.json`)

State lives on disk in the target repository (excluded via `.git/info/exclude` so `git diff` stays clean):

```json
{
  "project": "name",
  "roles": {
    "worker": { "providerInstanceId": "...", "model": "...", "options": {} },
    "default_reviewer": "self",
    "escalation_reviewer": { "providerInstanceId": "...", "model": "...", "options": {} }
  },
  "parallelism_cap": 3,
  "default_verify_cmd": "npm test",
  "baseline_notes": null,
  "tasks": [
    {
      "id": 1,
      "title": "...",
      "spec": "...",
      "status": "pending | in_progress | awaiting_review | awaiting_escalation | blocked | failed | done",
      "depends_on": [],
      "files": ["src/example.ts"],
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

## Installation

Symlink each skill folder into `~/.agents/skills` from the root of this repository (recommended so edits stay in sync):

```sh
mkdir -p ~/.agents/skills
ln -s "$(pwd)/orchestrator" ~/.agents/skills/orchestrator
ln -s "$(pwd)/worker"       ~/.agents/skills/worker
ln -s "$(pwd)/reviewer"     ~/.agents/skills/reviewer
```

Or copy them instead:

```sh
mkdir -p ~/.agents/skills
cp -r orchestrator worker reviewer ~/.agents/skills/
```

`worker` and `reviewer` set `disable-model-invocation: true` so they do not consume context or auto-trigger during ordinary chats; the orchestrator points delegated children directly to their file paths under `~/.agents/skills/`.

## Notes

- **Language:** Delegated briefs, state files, and review blocks stay in English for reliability; human-facing messages (role confirmation, checkpoints, final summary) follow the user's language.
- **State hygiene:** `.workflow/` is excluded locally via `.git/info/exclude` so workflow tracking never pollutes diffs or commits.
- **Fresh sessions:** Even when the Escalation Reviewer uses the same model as the Orchestrator, it always runs in a fresh `delegate_task` with narrow scope and no orchestrator conversation history.
