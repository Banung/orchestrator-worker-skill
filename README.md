# Orchestrator–Worker Skills

Two personal skills I made for my own AI coding workflow. This repository stores the skills and explains how I use them.

## What's included

- [orchestrator](orchestrator/SKILL.md): reads the repository context, decides on an approach, and prepares a `WORK ORDER`. Its role is read-only: it does not edit files or run tests/builds.
- [worker](worker/SKILL.md): carries out the work order within its scope, validates the results, and returns a `WORKER REPORT`.

The skills are used in separate chats. Work orders and reports are copied and pasted manually; there is no automatic communication between models.

## How to use

1. Activate `orchestrator` in the first chat and describe the task.
2. Copy the entire resulting `WORK ORDER` into a second chat using `worker`.
3. The worker carries out the task and returns a `WORKER REPORT` with the results, validation, and any blockers.
4. Review the results yourself, or copy the report back to the orchestrator if you want a review.

If there is not enough evidence to decide on an implementation, the orchestrator can issue a `DISCOVERY` task: gathering information without changing files or persistent state. An `IMPLEMENTATION` task allows changes within the boundaries specified in the work order.

Work orders and reports use English; the surrounding conversation follows the user's language.

## Installation

Copy the `orchestrator` and `worker` folders into the skills directory supported by your application. For applications that support `~/.agents/skills`, run these commands from the root of this repository:

```sh
mkdir -p ~/.agents/skills
cp -R -i orchestrator ~/.agents/skills/
cp -R -i worker ~/.agents/skills/
```

The `-i` option asks for confirmation before overwriting existing files. Reload the skills as required by your application. Installation paths and frontmatter support may differ between applications.

## Notes

These are instructions for the model, not tool permission controls. To keep the orchestrator strictly read-only, disable tools that can modify files or state in your application. The worker's output and validation claims still need to be checked.
