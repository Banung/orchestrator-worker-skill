# Multi-Agent Code Workflow Skills

Three skills for an automated, multi-model AI coding workflow: one model plans and reviews, a cheaper model writes the code, and a separate deep-review call is made only for the risky parts. This repository stores the skills and explains how the pieces fit together.

## What's included

- [orchestrator](orchestrator.md): plans the whole project once, keeps a persistent project state, schedules tasks by dependency, delegates them, routes results to review, handles retries, and writes the final summary. It does not write production code.
- [worker](worker.md): stateless. Receives **one** task spec plus project context, implements it, and returns compact search/replace edits plus a one-paragraph summary.
- [reviewer](reviewer.md): two roles in one file. **Part A** is the default review (plan conformance only). **Part B** is the escalation review (security, concurrency, deserialisation, database migrations, performance).

## Models and roles

| Role | Model | Skill |
|---|---|---|
| Orchestrator + default reviewer | Gemini 4 Argon | `orchestrator.md`, `reviewer.md` Part A |
| Worker | DeepSeek V4.1 Flash 1017 | `worker.md` |
| Escalation reviewer | Gemini 4 Argon (separate call, small fresh context) | `reviewer.md` Part B |

The skills are written by role, not by model name, so you can swap models by editing this table and your harness config. This setup targets a Google AI Pro subscription, where Gemini 4 Argon is available and Gemini 4 Carbon (AI Ultra) is not. If you later get Carbon, use it for the orchestrator row and keep Argon as the escalation reviewer; no skill file needs to change.

The orchestrator and the escalation reviewer run on the same model, so the escalation call must be a **separate call with its own small context** (see `reviewer.md` Part B). Its value comes from the narrow slice, the deep-review prompt, and independence from the orchestrator's reasoning, not from a stronger model.

## How it works

```
Orchestrator           ->  Worker         ->  Review            ->  pass          -> next task
 plan, state,               one task,          plan conformance     flagged       -> back to worker (max 2 retries)
 delegate                   edits + summary    only                 escalated     -> Escalation review -> pass / fail
```

1. **Plan (turn 1, once).** The orchestrator decomposes the whole project into an ordered task list with dependencies, acceptance criteria, expected files and risk tags.
2. **Each later turn.** The orchestrator reads the state, reviews any returned worker output, updates state, and delegates every task whose dependencies are `done`. Independent tasks (no shared files) run in parallel.
3. **Worker.** Implements exactly one task and returns edits, a summary, deviations, a self-reported sensitivity tag, and a status (`done` or `blocked` with a question).
4. **Review.** The reviewer (the orchestrator model) checks the output against its own plan and acceptance criteria. It does not hunt for runtime bugs or security flaws.
5. **Escalation.** The escalation reviewer is called only for tasks tagged `auth`, `concurrency`, `deserialisation` or `db_migration`, or when the orchestrator flags a concrete deep concern. Typically 5-10% of tasks. It receives a small slice (the orchestrator's summary, the flagged hunk, the task spec; about 50K tokens max), never the full codebase.
6. **Retries and humans.** A failed review goes back to the worker with only the specific issues. After 2 failed retries the task is `blocked` and the orchestrator asks you what to do (retry with guidance, skip, or amend the plan).
7. **Done.** When every task is `done` or `failed`, the orchestrator writes a final summary and the run ends.

### Review results

| Result | Meaning | Next step |
|---|---|---|
| `pass` | All criteria met, no conformance issues | task `done` |
| `flagged` | Conformance failure | retry worker with the listed issues |
| `escalated` | Conformance is clean and deep review is required | send to the escalation reviewer |

## Project state

The orchestrator works from a persistent state object that the harness re-sends every turn:

```json
{
  "project": "name",
  "tasks": [
    {
      "id": 1, "title": "...", "spec": "...",
      "status": "pending | in_progress | blocked | failed | done",
      "depends_on": [], "acceptance_criteria": ["..."],
      "risk_tags": [], "files": [],
      "worker_output": "diff + summary",
      "review_result": null, "retry_count": 0, "feedback": null
    }
  ],
  "current_task": 2, "completed": [1], "blocked": [], "escalations": []
}
```

To maximise prompt-cache hits, the harness assembles every prompt in a fixed order: skill and project spec (frozen), plan (frozen), ledger of completed tasks (append-only), live state (the only mutable part), then the current turn input. The orchestrator emits small `<state_patch>` and `<delegate>` blocks instead of re-sending state or context.

## Message formats

| Block | Emitted by | Purpose |
|---|---|---|
| `<plan>` | orchestrator | one-time task list |
| `<delegate>` | orchestrator | one task for the worker (spec, files, criteria, feedback on retries) |
| `<state_patch>` | orchestrator | state changes only |
| `=== EDIT / NEW / DELETE / SUMMARY / DEVIATIONS / SENSITIVE / STATUS` | worker | search/replace edits and report |
| `<review>` | orchestrator | `pass` / `flagged` / `escalated` with specific issues |
| `<escalation_verdict>` | escalation reviewer | `pass` / `fail` / `need_context` with findings |

The worker uses search/replace blocks instead of unified diffs: they are smaller and do not depend on line numbers, which models often get wrong.

Machine blocks and instructions to other models are in English. Messages to the human (questions, checkpoints, final summary) follow the user's language.

## What you need to provide (the harness)

The skills define behaviour, not plumbing. You need a script or app that:

- builds each prompt in the cache-friendly order above and puts the `<delegate>` block last in the worker prompt;
- calls the models (one call per delegated task, in parallel when allowed) and parses the blocks;
- applies `<state_patch>` and fills in `worker_output`;
- applies the worker's edits to the working tree, rejecting any edit whose SEARCH text is missing or not unique, and keeps the cumulative diff per task for review;
- builds the escalation reviewer's input from the orchestrator's summary, the flagged hunk and the task spec, and enforces the ~50K token cap;
- enforces the retry cap and the parallelism cap (default 3), and relays human questions and answers.

## Installation

Load each skill as the system prompt for its model. For `reviewer.md`, give the orchestrator **Part A** and the escalation reviewer **Part B** to save tokens (each part is self-contained).

If your application uses a skills directory instead, copy each file into its own folder as `SKILL.md`. For applications that support `~/.agents/skills`, run this from the root of this repository:

```sh
mkdir -p ~/.agents/skills/orchestrator ~/.agents/skills/worker ~/.agents/skills/reviewer
cp -i orchestrator.md ~/.agents/skills/orchestrator/SKILL.md
cp -i worker.md ~/.agents/skills/worker/SKILL.md
cp -i reviewer.md ~/.agents/skills/reviewer/SKILL.md
```

The `-i` option asks for confirmation before overwriting existing files. In this layout the whole `reviewer` file loads for both roles, which works because each role reads only its own part. Installation paths and frontmatter support may differ between applications.

## Notes

- These are instructions for the models, not enforcement. Caps (retries, escalation input size, file-overlap checks) and permissions should also be enforced by the harness.
- The orchestrator is meant to stay out of the code. Give it no file-writing or build tools.
- Worker self-reports (`SUMMARY`, `SENSITIVE`) are hints. The orchestrator verifies them against the diff, and you should still check the final results and run your own tests.
- Skill text is in English for the models. Adjust the language rule if you want a different default for human-facing messages.
