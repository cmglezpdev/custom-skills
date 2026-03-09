# Project Type: Monolith

Guidance for documenting monolith projects (single deployable application).

## Key Differences

Monoliths are simpler to document because there's one codebase, one deployment, and one mental model. The risk is under-documenting — since "everyone knows how it works" until someone new joins or the original team rotates.

## Structure Adjustments

### No `modules/` directory needed

Feature docs go directly in `product/` or a flat `features/` directory:

```
docs/
├── product/
│   ├── glossary.md
│   ├── user-auth.md
│   ├── billing.md
│   └── notifications.md
```

### Simpler conventions

If backend-only or frontend-only, `conventions/` can be a single file instead of subdirectories:

```
docs/
├── conventions/
│   └── coding-standards.md     # Single file covering all conventions
```

Split into `backend/` and `frontend/` only if the monolith has both layers with distinct patterns.

### Operations is critical

A monolith has one deployment target. Document it well:

```
docs/
├── operations/
│   ├── environments.md
│   ├── release-process.md
│   ├── backups.md
│   ├── monitoring.md
│   └── incident-response.md
```

## Minimum Viable Docs

1. `docs/README.md` (index)
2. `docs/onboarding/local-setup.md`
3. `docs/architecture/overview.md`
4. `docs/architecture/database.md`
5. `docs/adr/001-first-decision.md`
6. `docs/operations/release-process.md`
7. `docs/product/` for the 2-3 most complex features
8. `docs/templates/adr-template.md`

## Tips

- Don't over-document obvious code — focus on business rules, decisions, and operations
- `architecture/overview.md` is the most important doc; keep it current
- Every deploy-related change should update `operations/`
- Use ADRs to capture why the monolith is structured the way it is
