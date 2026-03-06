# VitePress Integration

Details on how the docs site is wired up with VitePress. For general VitePress knowledge (theming, custom components, deployment, advanced Markdown features), use the **vitepress** skill instead.

## Sidebar Configuration

The sidebar is **manually configured** in `apps/docs/.vitepress/config.ts`. New pages do not appear automatically — you must add them.

The sidebar is organized by path prefix. Each prefix has its own sidebar group:

| Prefix          | Contains                          |
| --------------- | --------------------------------- |
| `/architecture/`| Architecture docs                 |
| `/adr/`         | Architecture Decision Records     |
| `/modules/`     | All module documentation          |
| `/workflows/`   | Engineering & operations workflows|
| `/templates/`   | Doc templates                     |
| `/legacy/`      | Superseded docs                   |
| `/`             | Home + Registry                   |

### Adding a page to the sidebar

Find the matching prefix section in the `sidebar` object and add an entry:

```ts
// Example: adding a new feature doc to the identity module
{
  text: "Identity Module",
  items: [
    { text: "Overview", link: "/modules/identity/" },
    // ... existing items ...
    { text: "My New Feature", link: "/modules/identity/features/my-new-feature" },  // <-- add here
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

Top navigation is configured in `themeConfig.nav` in the same config file. You rarely need to change this — only when adding a major new top-level section.

## Plugins & Features

- **Mermaid diagrams**: Enabled via `vitepress-plugin-mermaid`. Use ` ```mermaid ` fenced code blocks.
- **Last updated**: `lastUpdated: true` — VitePress shows git-based last-updated dates on each page.
- **Search**: Local search provider (`search: { provider: "local" }`), no external service.

## Config File Location

`apps/docs/.vitepress/config.ts`
