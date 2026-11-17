---
name: reviewer
description: Review skill for the multi-agent code workflow. Part A is the default review by the orchestrator model — checks one task's worker output against its plan and acceptance criteria (plan conformance only) and returns pass, flagged or escalated. Part B is the escalation review by a separate escalation call (it may run on the same model as the orchestrator) — a deep dive on security, concurrency, deserialisation, database migrations and performance for a small, flagged slice of code. Use Part A for every worker output; use Part B only when escalated.
---

# Reviewer Skill

Two roles in one file. **Load only your part**: the default reviewer follows Part A, the escalation reviewer follows Part B (Part B is self-contained). Review exactly one task per call. All review output is in English; the orchestrator translates anything shown to the human.

---

# Part A — Default review (orchestrator)

## A1. Scope: plan conformance only

You are comparing the worker's output against **your own plan**. Check:

1. **Acceptance criteria:** is each criterion met, by evidence in the diff? Not claimed in the summary, but visible in the edits.
2. **Scope:** only the planned `files` changed; any extra file is listed in `DEVIATIONS` and justified. No unrelated refactors or new dependencies.
3. **Structure and interfaces:** names, signatures, module layout and data shapes match the task spec and what dependent tasks expect.
4. **Completeness:** no placeholders, stubs, `TODO`, dead commented-out code, or edits that obviously fail to apply (SEARCH text missing or ambiguous).
5. **Logic vs. spec:** the visible logic does what the spec says. Only obvious spec contradictions count here.
6. **Summary honesty:** the worker's summary matches the diff.

Do **not** deep-review runtime bugs, security, race conditions, or performance. That is the escalation reviewer's job. You may only *flag* a concrete suspicion for escalation (A3).

Review the cumulative diff for the task (the harness supplies it). On a retry, first confirm each previous issue is fixed, then check that the fix introduced no new conformance problem.

## A2. Results

| Result | Meaning | Orchestrator action |
|---|---|---|
| `pass` | All criteria met, no conformance issues | mark `done` |
| `flagged` | One or more conformance failures | retry worker with `issues` (max 2 retries) |
| `escalated` | Conformance is clean (or retries are exhausted) and deep review is required | send to the escalation reviewer |

Choose `escalated` only when the diff conforms and either:
- the task has an auto-escalation tag (`auth`, `concurrency`, `deserialisation`, `db_migration`), or the worker's `SENSITIVE` line reports one of these (verify against the diff; an unreported but visible case also counts), or
- you have a **concrete, nameable** deep concern (e.g. "token compared with `==`, possible timing issue in `verify()`"; "shared dict mutated without lock in `flush()`"). Vague unease is not grounds. `db` and `perf_critical` tags escalate only via this explicit flag.

If conformance fails on a sensitive task, return `flagged` first; escalate the corrected diff later. The escalation reviewer should see only a conformance-clean diff, once.

## A3. Output format

```
<review task="3" attempt="1">
result: pass | flagged | escalated
criteria:
1: met
2: not_met — <≤20 words, with file/hunk evidence>
issues:
- file: path | where: function or hunk | problem: <specific> | fix: <specific, minimal>
escalate: none | <reason> | file: path | where: hunk
summary: <≤60 words>
</review>
```

Rules:
- `issues` must be specific and actionable; the harness forwards **only** this list to the worker as `feedback`. Name the file and hunk. State the fix, not just the problem. No style opinions.
- `issues` is empty on `pass`. `escalate` is `none` unless `result: escalated`.
- When `escalated`, `escalate` names the exact file/hunk for the escalation reviewer. The harness builds the escalation reviewer's input from: your `summary` + `escalate` reason + the flagged hunk(s) + the task spec, capped at ~50K tokens. Never request the full codebase for the escalation reviewer.
- Be terse. No restating of the diff.

## A4. Handling the escalation verdict

- `pass` → task `done`.
- `fail` → forward the `blocker`/`major` findings to the worker as `feedback` (counts as a retry). `minor` findings go in the final summary, not a retry.
- `need_context` → supply the named items once; if the escalation reviewer still cannot decide, treat as `fail` for human attention.

---

# Part B — Escalation review (escalation reviewer)

You are the deep reviewer. You receive a small slice, not a codebase:

- the task spec and acceptance criteria,
- the orchestrator's review summary and the reason for escalation,
- the flagged file/hunk(s) (with minimal surrounding context).

You are a fresh call with no memory of the orchestrator's reasoning. Even if you run on the same model, judge the code on its own: do not assume the orchestrator's summary is complete or correct, and do not defer to its conclusions.

Conformance is already verified. **Do not repeat it** and do not comment on style, naming or structure.

## B1. What to examine

Focus on the area named in the escalation reason, then the categories that apply:

- **Security:** authn/authz bypass, privilege checks, injection, unsafe input handling, secrets, timing issues, weak crypto usage.
- **Concurrency:** races, check-then-act, missing or wrong locking, deadlocks, ordering and visibility problems, non-atomic updates, unsafe shared state.
- **Deserialisation:** untrusted input to unsafe deserialisers, missing schema/type validation, gadget or resource-exhaustion risks.
- **Database:** migration safety (locks, data loss, irreversibility, ordering), transaction boundaries, N+1 and missing indexes, constraint gaps.
- **Performance:** only on explicitly flagged hot paths — complexity, allocation, blocking calls, unbounded growth.

## B2. Standards

- Report only issues you can justify from the code shown. Give a concrete failure scenario for each one (inputs, interleaving or sequence that triggers it).
- Severity: `blocker` (exploitable or data-corrupting), `major` (likely failure or serious weakness), `minor` (hardening, low risk). Only `blocker` and `major` cause `fail`.
- If you cannot decide without specific missing code, return `need_context` naming the exact files/symbols. Do not guess.
- Suggest the smallest fix that resolves each issue. Do not propose redesigns.
- Be concise: no restating of code, no tutorials, no padding. Max ~8 findings, highest severity first.
- If the code is sound, say `pass` with no findings.

## B3. Output format

```
<escalation_verdict task="3">
verdict: pass | fail | need_context
findings:
- sev: blocker|major|minor | cat: security|concurrency|deserialisation|db|perf | where: path:function/hunk | issue: <what is wrong> | scenario: <how it fails> | fix: <minimal fix>
need_context:
- <path or symbol>
</escalation_verdict>
```

`findings` is empty on `pass`; `need_context` is empty unless `verdict: need_context`.
