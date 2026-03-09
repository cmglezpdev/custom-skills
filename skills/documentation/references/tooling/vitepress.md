# Tooling Adapter: VitePress

Optional adapter for projects using VitePress as their docs site. Read this only if the project has a `.vitepress/` directory.

## Sidebar Configuration

The sidebar is **manually configured** in `.vitepress/config.ts`. New pages do not appear automatically — you must add them.

The sidebar is organized by path prefix. Each prefix has its own sidebar group:

| Prefix | Contains |
| --- | --- |
| `/` | Home + Registry/Index |
| `/onboarding/` | New developer guides |
| `/architecture/` | Architecture docs |
| `/adr/` | Architecture Decision Records |
| `/product/` | Business-facing docs, PRDs |
| `/conventions/` | Backend and frontend coding conventions |
| `/modules/` | All module documentation (monorepo) |
| `/operations/` | Operational procedures and incident response |
| `/security/` | Security policies and posture docs |
| `/workflows/` | Engineering processes |
| `/templates/` | Doc templates |
| `/legacy/` | Superseded docs |

### Adding a page to the sidebar

Find the matching prefix section in the `sidebar` object and add an entry:

```ts
// Example: adding a new feature doc
{
  text: "Identity Module",
  items: [
    { text: "Overview", link: "/modules/identity/" },
    // ... existing items ...
    { text: "My New Feature", link: "/modules/identity/features/my-new-feature" },  // <-- add here
  ],
},
```

### Adding a new section to the sidebar

Add a new key in the `sidebar` object:

```ts
sidebar: {
  // ... existing sections ...
  '/onboarding/': [
    {
      text: "Onboarding",
      items: [
        { text: "Local Setup", link: "/onboarding/local-setup" },
        { text: "Project Structure", link: "/onboarding/project-structure" },
        { text: "Coding Standards", link: "/onboarding/coding-standards" },
      ],
    },
  ],
  '/security/': [
    {
      text: "Security",
      items: [
        { text: "Secrets Management", link: "/security/secrets-management" },
        { text: "Auth Overview", link: "/security/auth-overview" },
        { text: "Data Protection", link: "/security/data-protection" },
      ],
    },
  ],
},
```

### Adding a new module to the sidebar

Add a new group object inside the `/modules/` sidebar array:

```ts
{
  text: "New Module",
  items: [
    { text: "Overview", link: "/modules/new-module/" },
  ],
},
```

## Nav Bar

Top navigation is configured in `themeConfig.nav`. When the number of top-level sections exceeds 6-7, group related items into dropdown menus:

```ts
nav: [
  { text: "Home", link: "/" },
  { text: "Registry", link: "/registry" },
  {
    text: "Guides",
    items: [
      { text: "Onboarding", link: "/onboarding/local-setup" },
      { text: "Conventions", link: "/conventions/backend/" },
      { text: "Workflows", link: "/workflows/engineering/overview" },
    ],
  },
  {
    text: "Reference",
    items: [
      { text: "Architecture", link: "/architecture/overview" },
      { text: "ADR", link: "/adr/" },
      { text: "Modules", link: "/modules/identity/" },
    ],
  },
  {
    text: "Operations",
    items: [
      { text: "Operations", link: "/operations/" },
      { text: "Security", link: "/security/" },
    ],
  },
],
```

## Plugins & Features

- **Mermaid diagrams**: Use ` ```mermaid ` fenced code blocks (requires `vitepress-plugin-mermaid`)
- **Last updated**: `lastUpdated: true` shows git-based dates on each page
- **Search**: Local search provider (`search: { provider: "local" }`), no external service needed

## Config File Location

Look for `.vitepress/config.ts` relative to the docs directory (e.g., `apps/docs/.vitepress/config.ts` in a monorepo).
