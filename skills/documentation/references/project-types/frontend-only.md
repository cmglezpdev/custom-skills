# Project Type: Frontend-Only

Guidance for documenting frontend-only projects (React, Next.js, Vue, Angular, etc.).

## Key Differences

Frontend projects are UI-centric. Documentation should emphasize component architecture, state management, design patterns, and user-facing features. Operations is typically lighter (static hosting, CDN).

## Structure Adjustments

### Skip backend conventions

Remove or leave empty `conventions/backend/`. Focus on:

```
docs/
├── conventions/
│   └── frontend/
│       ├── routing.md
│       ├── state-management.md
│       ├── design-system.md
│       ├── forms-and-validation.md
│       └── component-patterns.md
```

### Product docs are important

Feature specs should include UI flows, wireframes references, and user interaction patterns:

```
docs/
├── product/
│   ├── glossary.md
│   ├── dashboard.md           # Feature with UI flow description
│   └── settings-panel.md
```

### Lighter operations

```
docs/
├── operations/
│   ├── environments.md        # Dev, staging, prod URLs
│   ├── release-process.md     # Build + deploy pipeline
│   └── monitoring.md          # Error tracking, analytics
```

### Architecture focuses on UI concerns

```
docs/
├── architecture/
│   ├── overview.md            # Component hierarchy, data flow
│   ├── api-integration.md     # How the frontend talks to backends
│   └── auth-flow.md           # Token handling, session management
```

### Security focuses on client-side

```
docs/
├── security/
│   ├── auth-flow.md           # Token storage, refresh logic
│   ├── xss-prevention.md      # Sanitization, CSP
│   └── data-handling.md       # PII in localStorage, cookies
```

## Minimum Viable Docs

1. `docs/README.md`
2. `docs/onboarding/local-setup.md`
3. `docs/architecture/overview.md`
4. `docs/conventions/frontend/` (at least component-patterns and state-management)
5. `docs/product/` for the 2-3 core features
6. `docs/operations/release-process.md`

## Tips

- Document component naming conventions and file structure early
- State management decisions deserve an ADR
- Design system docs can link to Storybook or Figma instead of duplicating
- API integration patterns (error handling, retries, caching) belong in `conventions/frontend/`
