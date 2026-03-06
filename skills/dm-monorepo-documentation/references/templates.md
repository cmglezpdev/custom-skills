# Documentation Templates

Read this file when you need the full template for a specific document type. Pick the template that matches your document type from the decision guide in SKILL.md.

---

## Feature Template

Use for new features, capabilities, or domain logic documentation.

```md
---
title: '<Feature name>'
summary: '<What this feature does>'
status: active
owner: '<team>'
last_reviewed: YYYY-MM-DD
source_paths:
  - '<path>'
---

# [Feature name]

## Summary

What this feature does and who uses it.

## Business Rules

- Rule 1
- Rule 2

## API/Contract

- Endpoints/events/commands
- Request and response shape references

## Data Model Impact

- Entities/tables/indices affected

## Failure Modes

- Known errors and fallback behavior

## Observability

- Metrics
- Logs
- Traces

## Change Log

- YYYY-MM-DD: short change entry

## Source Paths

- `apps/api/src/...`
```

---

## ADR Template

Use for architecture decisions. Number sequentially — check `apps/docs/adr/` for the latest number.

```md
---
title: '<NNN - Decision Title>'
summary: '<Short decision summary>'
status: active
owner: '<team>'
last_reviewed: YYYY-MM-DD
source_paths:
  - '<path>'
---

# NNN - [Decision title]

## Status

Proposed | Accepted | Superseded

## Date

YYYY-MM-DD

## Context

Describe the problem, constraints, and forces.

## Decision

Describe the decision and scope.

## Alternatives Considered

- Alternative A
- Alternative B

## Consequences

- Positive outcomes
- Tradeoffs and risks

## Related Docs

- Feature docs:
- Domain docs:
- Runbooks:

## Source Paths

- `apps/api/src/...`
```

---

## Workflow Template

Use for engineering or operations processes.

```md
---
title: '<Workflow name>'
summary: '<What this workflow accomplishes>'
status: active
owner: '<team>'
last_reviewed: YYYY-MM-DD
source_paths:
  - '<path>'
---

# [Workflow name]

## Objective

What this workflow accomplishes.

## Preconditions

- Access needed
- Environment needed

## Steps

1. Step one.
2. Step two.

## Validation

- Expected outputs/checks.

## Rollback / Recovery

- How to revert if needed.

## Related Runbooks

- Link runbooks.

## Source Paths

- `apps/...`
```

---

## Runbook Template

Use for incident response, operational procedures, and troubleshooting guides.

```md
---
title: '<Runbook name>'
summary: '<When to use this runbook>'
status: active
owner: '<team>'
last_reviewed: YYYY-MM-DD
source_paths:
  - '<path>'
---

# [Runbook name]

## Trigger

When this runbook should be used.

## Symptoms

- Symptom 1
- Symptom 2

## Diagnostics

1. Check X.
2. Check Y.

## Mitigation

1. Immediate mitigation step.
2. Stabilization step.

## Recovery

1. Recovery step.
2. Validation step.

## Post-Incident

- Follow-up tasks.
- Documentation updates required.

## Source Paths

- `infra/...`
```
