# Project Type: Monorepo

Guidance for documenting monorepo projects (multiple packages/apps in one repository).

## Key Differences

Monorepos have more surface area. The main risk is docs sprawl — too many docs in too many places with no discoverability.

## Structure Adjustments

### Add `modules/` directory

The standard `docs/` tree gains a `modules/` directory for per-module documentation:

```
docs/
├── modules/
│   └── <module-name>/
│       ├── index.md          # Module overview (entry point)
│       ├── features/         # What this module does
│       │   ├── overview.md
│       │   └── feature-name.md
│       ├── domain/           # Business rules, entities, invariants
│       │   └── topic.md
│       └── runbooks/         # Module-specific operational guides
│           └── runbook-name.md
```

Each module should have at minimum an `index.md` that explains:
- What the module does
- Its public API or integration points
- Links to its feature, domain, and runbook docs

### Registry file

A `registry.md` at the docs root is **strongly recommended** for monorepos. It serves as a canonical index of all documentation files, their status, and ownership. Without it, docs become hard to discover as the count grows past 20-30 files.

Format:

```md
| Path | Type | Status | Owner | Notes |
|------|------|--------|-------|-------|
| modules/identity/features/auth.md | Feature | active | Backend | ... |
```

### Conventions are critical

With multiple packages sharing patterns, `conventions/backend/` and `conventions/frontend/` become essential. Document shared patterns once here rather than repeating in each module.

## What Goes Where

- **Module-specific feature** → `modules/<module>/features/`
- **Cross-module architecture** → `architecture/`
- **Shared coding pattern** → `conventions/backend/` or `conventions/frontend/`
- **Decision affecting multiple modules** → `adr/`
- **Module-specific operational guide** → `modules/<module>/runbooks/`
- **Project-wide operational guide** → `operations/`

## Minimum Viable Docs

1. `docs/README.md` (index)
2. `docs/registry.md`
3. `docs/onboarding/local-setup.md`
4. `docs/architecture/overview.md`
5. `docs/modules/<main-module>/index.md` for each active module
6. `docs/adr/001-first-decision.md`
7. `docs/conventions/backend/` or `conventions/frontend/` (at least one)

## Tips

- Use the registry as the source of truth for what exists
- Each module should own its own docs — the module maintainer updates them
- Cross-cutting docs (architecture, ADRs, operations) need a shared owner
- Consider a docs site tool (VitePress, Docusaurus) once you exceed 30 docs — see `references/tooling/` for adapters
