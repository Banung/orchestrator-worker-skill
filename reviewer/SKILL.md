---
name: reviewer
description: Two-stage review skill for the orchestrator-worker workflow (conformance review and escalation deep review).
disable-model-invocation: true
---

# Reviewer Skill (Router)

This skill separates two distinct review roles into isolated reference files so neither role leaks into the other's context. **Read only the file matching your assigned role:**

- **Default Reviewer (plan conformance):** Read [conformance.md](conformance.md) and follow it exclusively.
- **Escalation Reviewer (deep audit — security, concurrency, deserialisation, DB, flagged perf):** Read [escalation.md](escalation.md) and follow it exclusively.

If your prompt does not state which review role you are executing, ask the caller before proceeding.
