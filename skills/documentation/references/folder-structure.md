# Documentation Folder Structure

Standard `docs/` directory tree for any project. Adapt based on your project type — see `references/project-types/` for specific guidance.

```
docs/
├── README.md                    # Docs index — "if you want X, read Y"
├── changelog.md                 # Notable project-wide changes (optional)
│
├── onboarding/                  # New developer guides
│   ├── local-setup.md           # How to set up the dev environment
│   ├── project-structure.md     # Repo layout and key directories
│   └── coding-standards.md      # General coding conventions
│
├── architecture/                # How the system works
│   ├── overview.md              # High-level architecture map
│   ├── database.md              # Schema design, relationships
│   └── integrations.md          # External services, APIs
│
├── adr/                         # Architecture Decision Records
│   ├── index.md                 # ADR index page (optional)
│   └── NNN-decision-title.md    # Individual ADRs (sequential numbering)
│
├── product/                     # Business-facing documentation
│   ├── glossary.md              # Domain terminology
│   └── <feature-name>.md        # Product feature specs / PRDs
│
├── conventions/                 # Coding standards by layer
│   ├── backend/                 # Backend conventions (if applicable)
│   │   ├── api-conventions.md
│   │   ├── error-handling.md
│   │   └── module-patterns.md
│   └── frontend/                # Frontend conventions (if applicable)
│       ├── component-patterns.md
│       ├── state-management.md
│       └── design-system.md
│
├── modules/                     # Per-module docs (monorepo only)
│   └── <module-name>/
│       ├── index.md             # Module overview (entry point)
│       ├── features/            # What this module does
│       │   ├── overview.md
│       │   └── feature-name.md
│       ├── domain/              # Business rules, entities, invariants
│       │   └── topic.md
│       └── runbooks/            # Module-specific operational guides
│           └── runbook-name.md
│
├── operations/                  # Deployment, monitoring, incidents
│   ├── environments.md          # Dev, staging, production setup
│   ├── release-process.md       # How to ship
│   ├── backups.md               # Backup strategy and restoration
│   ├── monitoring.md            # Alerting, dashboards, health checks
│   └── incident-response.md     # What to do when things break
│
├── security/                    # Security posture
│   ├── secrets-management.md    # How secrets are stored and rotated
│   ├── auth-overview.md         # Cross-cutting auth architecture
│   ├── data-protection.md       # Encryption, PII handling
│   └── threat-model.md          # Known risks and mitigations
│
├── workflows/                   # Step-by-step processes
│   ├── engineering/             # Development workflows
│   │   ├── overview.md
│   │   └── workflow-name.md
│   └── operations/              # Operational workflows (optional)
│       └── workflow-name.md
│
├── templates/                   # Reference templates (copy, don't edit)
│   ├── adr-template.md
│   ├── feature-template.md
│   ├── workflow-template.md
│   ├── runbook-template.md
│   ├── postmortem-template.md
│   ├── product-template.md
│   └── onboarding-template.md
│
└── legacy/
    └── superseded/              # Old docs replaced by newer versions
```

## Section Reference

| Section | Purpose | Required? |
|---------|---------|-----------|
| `onboarding/` | Get new developers productive in 1 day | Yes |
| `architecture/` | Mental models — how the system thinks | Yes |
| `adr/` | Why decisions were made, not just what | Yes (at least 1) |
| `product/` | Business-facing feature specs and glossary | Recommended |
| `conventions/` | Project-wide coding standards by layer | Recommended |
| `modules/` | Per-module docs (features, domain, runbooks) | Monorepo only |
| `operations/` | How to deploy, monitor, and recover | Yes for backends |
| `security/` | Security posture and threat mitigation | Recommended |
| `workflows/` | Step-by-step engineering processes | Optional |
| `templates/` | Starter templates for each doc type | Recommended |
| `legacy/` | Superseded docs preserved for history | Optional |

## Creating a New Section

1. Create the directory under `docs/`
2. Add an `index.md` or first document with proper frontmatter
3. Update `docs/README.md` with a link to the new section
4. Update registry/index if the project uses one
5. If using a docs site tool, update sidebar config (see `references/tooling/`)

## File Naming Conventions

Use predictable, descriptive names:

- `overview.md`, `local-setup.md`, `database.md`, `auth.md`
- `NNN-decision-title.md` for ADRs (sequential numbering)
- `feature-name.md` for features (kebab-case)

Avoid vague names: `notes.md`, `ideas.md`, `important.md`, `misc.md`
