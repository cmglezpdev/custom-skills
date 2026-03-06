# Documentation Folder Structure

Complete tree of the `apps/docs/` directory and what each part is for.

```
apps/docs/
├── .vitepress/config.ts        # Nav bar + sidebar (manually configured)
├── index.md                     # Docs portal home page
├── registry.md                  # Canonical index of ALL doc files
│
├── architecture/                # Cross-cutting architecture docs
│   ├── overview.md              # High-level architecture map
│   ├── api-module-architecture.md
│   └── documentation-structure.md
│
├── adr/                         # Architecture Decision Records
│   ├── index.md                 # ADR index page
│   └── NNN-decision-title.md   # Individual ADRs (sequential: 004, 005, 006...)
│
├── modules/                     # Module-centric documentation
│   └── <module-name>/
│       ├── index.md             # Module overview (entry point)
│       ├── features/            # Feature documentation
│       │   ├── overview.md      # Feature overview for the module
│       │   └── feature-name.md  # Individual feature docs
│       ├── domain/              # Business rules, entities, invariants
│       │   └── topic.md
│       └── runbooks/            # Operational guides & troubleshooting
│           ├── overview.md (or index.md)
│           └── runbook-name.md
│
├── workflows/
│   ├── engineering/             # Development workflows
│   │   ├── overview.md
│   │   └── workflow-name.md
│   └── operations/              # Operations workflows
│       ├── overview.md
│       └── workflow-name.md
│
├── templates/                   # Reference templates (do not edit these)
│   ├── adr-template.md
│   ├── feature-template.md
│   ├── workflow-template.md
│   └── runbook-template.md
│
└── legacy/
    └── superseded/              # Old docs replaced by newer versions
        └── overview.md
```

## Existing Modules

These modules already have documentation directories:

| Module        | Path                          | Description              |
| ------------- | ----------------------------- | ------------------------ |
| identity      | `modules/identity/`           | Auth, users, clients     |
| infra         | `modules/infra/`              | Dokploy, VPN, infra ops  |
| observability | `modules/observability/`      | OTEL, logging, metrics   |
| panel         | `modules/panel/`              | Admin panel              |
| packages      | `modules/packages/`           | Shared npm packages      |
| shared        | `modules/shared/`             | Cross-module services    |

## Creating a New Module's Docs

1. Create `modules/<new-module>/index.md` with proper frontmatter
2. Add subdirectories as needed: `features/`, `domain/`, `runbooks/`
3. Add the module section to `registry.md`
4. Add a sidebar group in `.vitepress/config.ts` under the `/modules/` section
