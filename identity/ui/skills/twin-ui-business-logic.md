# TWIN UI — Business Logic & Domain Patterns

You are working on the domain conventions of the TWIN Foundation UI component library — a dual-framework (React 19 + Svelte 5) component library published as `@twin.org/ui-*`.

## Domain Overview

This is a **pure UI component library** — not an application. It has no auth system, no API layer, and no global state. Business logic means: component behavior contracts, variant semantics, design token meaning, and publishing conventions.

## Component Behavior Contracts

### Color Semantics
Components must use these colors with consistent meaning across the library:

| Color | Semantic meaning | When to use |
|-------|-----------------|-------------|
| `primary` | Main brand action | Primary CTA, key actions |
| `secondary` | Supporting action | Secondary CTAs, alternatives |
| `error` | Destructive, danger | Delete, remove, irreversible |
| `warning` | Caution, confirm | Potentially risky, confirm prompts |
| `success` | Positive, complete | Confirmation, done states |
| `info` | Informational, neutral | Tips, status, read-only alerts |
| `plain` | Minimal styling | Low-emphasis actions |
| `ghost` | Transparent, border-only | Tertiary actions, toolbars |
| `dark` | Inverted | Dark backgrounds, inverse contexts |

### Size Semantics
| Size | Context |
|------|---------|
| `xs` | Dense UIs, tags, chips |
| `sm` | Compact forms, secondary actions |
| `md` | Default — most use cases |
| `lg` | Prominent actions, hero sections |
| `xl` | Marketing pages, large displays |

## Package Publishing Conventions

- All packages under `@twin.org/` scope
- Version: prerelease on `next` branch (`0.0.x-next.y`), stable on `main`
- Conventional Commits required — `release-please` auto-generates changelog
- Each package exports: ESM, CJS, TypeScript types, CSS, and individual component paths

### Package Export Contract
```json
{
  ".": "main bundle",
  "./<component>": "tree-shakeable per-component export",
  "./icons/<name>": "individual icon (lucide-react based)",
  "./css/<name>.css": "component-specific CSS",
  "./config/<name>.mjs": "Tailwind config utilities"
}
```

## Component Hierarchy

Components extend Flowbite for semantic/accessible base behavior, then layer TWIN design tokens on top.

```
HTML element (semantic)
  → Flowbite component (accessible, unstyled base)
    → TWIN component (design tokens applied, props extended)
      → Consumer application (className overrides via cn())
```

Consumers should not need to dig into Flowbite internals.

## Locale / i18n Contract (Svelte)

- Every user-visible string in a Svelte component goes into `locales/en.json`
- Key format: `componentName.propertyDescription` (e.g., `datepicker.clearButton`)
- The `merge-locales` build step merges package locales into consuming app locale files
- React components: strings passed as props (no built-in i18n — consuming app owns translation)

## Build Contract

Every package must produce:
1. `dist/types/index.d.ts` — TypeScript declarations
2. `dist/esm/index.mjs` — ES module bundle
3. `dist/cjs/index.cjs` — CommonJS bundle
4. `dist/css/` — PostCSS-processed Tailwind CSS

If any of these are missing, the package is broken. CI will fail.

## Dependency Rules

| Dependency | Policy |
|------------|--------|
| `flowbite`, `flowbite-react`, `flowbite-svelte` | Peer dependency — consuming app installs it |
| `@radix-ui/*` | Optional peer — only needed for specific components |
| `@twin.org/ui-tailwind` | Workspace dependency — always at `*` (latest workspace version) |
| `react`, `svelte` | Peer — never bundle the framework itself |
| `lucide-react` | Production dep in react package |

## Versioning & Release

- Branch `next`: prerelease versions (`0.0.3-next.6`)
- Branch `main`: stable releases
- `release-please` reads Conventional Commits and bumps versions automatically
- Never manually edit `CHANGELOG.md` — it's auto-generated
- Never manually bump versions in `package.json` — `release-please` handles this

## Apache-2.0 Compliance

Every source file must have:
```
// Copyright 2025 TWIN Foundation
// SPDX-License-Identifier: Apache-2.0
```

This is enforced by ESLint. PRs without this header on new files will fail CI.
