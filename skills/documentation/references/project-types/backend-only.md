# Project Type: Backend-Only

Guidance for documenting backend-only projects (APIs, services, workers — NestJS, Express, FastAPI, Rails, etc.).

## Key Differences

Backend projects are data and logic-centric. Documentation should emphasize API design, data models, integrations, error handling, and operations. There is no UI to document.

## Structure Adjustments

### Skip frontend conventions

Remove or leave empty `conventions/frontend/`. Focus on:

```
docs/
├── conventions/
│   └── backend/
│       ├── api-conventions.md       # Naming, versioning, response format
│       ├── error-handling.md        # Error codes, exception hierarchy
│       ├── validation.md            # Input validation patterns
│       ├── events-and-queues.md     # Async messaging patterns
│       └── module-patterns.md       # Code organization, DDD layers
```

### Operations is critical

Backend services need robust operational docs:

```
docs/
├── operations/
│   ├── environments.md
│   ├── release-process.md
│   ├── backups.md
│   ├── monitoring.md
│   ├── incident-response.md
│   └── database-migrations.md
```

### Architecture focuses on data and integrations

```
docs/
├── architecture/
│   ├── overview.md              # Module structure, data flow
│   ├── database.md              # Schema design, relationships
│   ├── integrations.md          # External APIs, third-party services
│   └── auth.md                  # Authn/authz architecture
```

### Security is comprehensive

```
docs/
├── security/
│   ├── secrets-management.md
│   ├── authz-authn.md           # Roles, permissions, token lifecycle
│   ├── data-protection.md       # Encryption, PII handling
│   ├── rate-limiting.md         # Abuse prevention
│   └── threat-model.md
```

## Minimum Viable Docs

1. `docs/README.md`
2. `docs/onboarding/local-setup.md`
3. `docs/architecture/overview.md`
4. `docs/architecture/database.md`
5. `docs/conventions/backend/api-conventions.md`
6. `docs/operations/release-process.md`
7. `docs/operations/monitoring.md`
8. `docs/adr/001-first-decision.md`
9. `docs/security/secrets-management.md`

## Tips

- API conventions should be documented before the second endpoint is built
- Every database schema change deserves a mention in `architecture/database.md`
- Error handling patterns prevent inconsistency across endpoints — document early
- Integration docs should include auth method, rate limits, and failure behavior
- Runbooks for common operational tasks save hours during incidents
