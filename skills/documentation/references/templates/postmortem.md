# Postmortem Template

Use for post-incident reports. Place in `operations/` or the relevant module's `runbooks/`.

```md
---
title: '<Incident title — YYYY-MM-DD>'
summary: '<One-line incident summary>'
status: active
owner: '<team>'
last_reviewed: YYYY-MM-DD
source_paths: []
---

# Incident: [Title] — [Date]

## Incident Summary

What happened, when, and the overall impact.

## Timeline

| Time (UTC) | Event |
|------------|-------|
| HH:MM | First alert / symptom detected |
| HH:MM | Investigation started |
| HH:MM | Root cause identified |
| HH:MM | Mitigation applied |
| HH:MM | Service fully restored |

## Root Cause

What caused the incident. Be specific — distinguish between trigger and underlying cause.

## Impact

- **Duration**: X hours
- **Users affected**: description
- **Data loss**: yes/no, details
- **SLA breach**: yes/no

## Mitigation Steps

What was done to stop the bleeding.

1. Step one.
2. Step two.

## Lessons Learned

- What went well
- What went poorly
- Where we got lucky

## Action Items

| Action | Owner | Due Date | Status |
|--------|-------|----------|--------|
| Action 1 | @person | YYYY-MM-DD | pending |
| Action 2 | @person | YYYY-MM-DD | pending |

## Related Docs

- Runbooks used:
- ADRs related:
```
