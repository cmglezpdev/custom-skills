---
name: docs
description: >
  How to create, update, and manage documentation in the DaraMex Monorepo VitePress docs app (apps/docs/).
  Use this skill whenever the user asks to add documentation, update docs, create a runbook, write an ADR,
  document a feature, add module docs, update the registry, or any documentation-related task in this project.
  Also trigger when making behavior-changing code changes (features, fixes, refactors, integrations, infra changes)
  that require accompanying documentation updates — even if the user doesn't explicitly mention "docs".
---

# DaraMex Documentation Skill

Guide for working with the DaraMex internal docs site (VitePress). For VitePress-specific help (theming, custom components, advanced Markdown), use the **vitepress** skill.

## Reference Files

Read these as needed — don't load everything upfront:

| File | When to read |
| ---- | ------------ |
| `references/templates.md` | When creating a new doc (has Feature, ADR, Workflow, Runbook templates) |
| `references/folder-structure.md` | When unsure where a doc goes or when adding a new module |
| `references/vitepress-integration.md` | When adding a page to the sidebar or touching VitePress config |

---

## Mandatory Workflow (every documentation change)

Follow these steps in order. Skipping any step risks orphaned pages, broken navigation, or stale registry entries.

### Step 1: Read the registry

Read `apps/docs/registry.md` before doing anything. Check if a doc already exists. Prefer updating existing docs over creating new ones.

### Step 2: Write or update the document

Every doc file must include this frontmatter:

```md
---
title: '<doc title>'
summary: '<one-line purpose>'
status: active
owner: '<team-or-role>'
last_reviewed: YYYY-MM-DD
source_paths:
  - '<repo/path/used/as/source>'
replaces: '<old doc path, if replacing>'
replaced_by: '<new doc path, if being replaced>'
---
```

- `status`: `draft` | `active` | `superseded` | `archived`
- `last_reviewed`: always set to today's date
- `replaces` / `replaced_by`: only when superseding another doc

For the document body, read `references/templates.md` to get the right template for your doc type.

### Step 3: Update the registry

Edit `apps/docs/registry.md`:

1. **New docs**: Add a row to the correct table section (`| Path | Type | Status | Owner | Notes |`)
2. **Superseded docs**: Change status to `superseded`, add note
3. **Recently Changed Docs**: Add a bullet at the bottom: `- YYYY-MM-DD: Short description`

### Step 4: Update the sidebar

New pages must be added to `apps/docs/.vitepress/config.ts`. Read `references/vitepress-integration.md` for details on which sidebar section to use and the exact format.

### Step 5: Handle superseded docs (only when replacing)

1. Move old doc to `apps/docs/legacy/superseded/`
2. Set `status: superseded` and add `replaced_by` in old doc's frontmatter
3. Add `replaces` in new doc's frontmatter
4. Update both entries in `registry.md`

---

## Document Type Decision Guide

| Change type              | Location                                   | Template |
| ------------------------ | ------------------------------------------ | -------- |
| New module feature       | `modules/<module>/features/<name>.md`      | Feature  |
| Domain / business logic  | `modules/<module>/domain/<name>.md`        | Feature  |
| Architecture decision    | `adr/NNN-<title>.md`                       | ADR      |
| Engineering workflow     | `workflows/engineering/<name>.md`          | Workflow |
| Operations workflow      | `workflows/operations/<name>.md`           | Workflow |
| Infra runbook            | `modules/infra/runbooks/<name>.md`         | Runbook  |
| Observability runbook    | `modules/observability/runbooks/<name>.md` | Runbook  |
| Module-specific runbook  | `modules/<module>/runbooks/<name>.md`      | Runbook  |
| Architecture overview    | `architecture/<name>.md`                   | None     |
| Minor update             | Edit in place, update `last_reviewed`      | N/A      |

---

## Quick Checklist

Before submitting any documentation change:

- [ ] Read `registry.md` first
- [ ] Frontmatter is complete with today's date in `last_reviewed`
- [ ] Registry updated (new row + "Recently Changed Docs" entry)
- [ ] Sidebar updated in `.vitepress/config.ts` (new pages only)
- [ ] Superseded docs handled properly (if replacing)
