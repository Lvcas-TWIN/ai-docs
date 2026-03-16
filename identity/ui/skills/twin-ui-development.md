# TWIN UI — Development Rules & Patterns

You are working on the TWIN Foundation UI component library — a monorepo with React 19 + Svelte 5 components published as `@twin.org/ui-*` packages.

## Before Writing Any Code

1. Read `CLAUDE.md` at the repo root for current rules
2. Check if the task touches React (`packages/ui-components-react/`), Svelte (`packages/ui-components-svelte/`), or design tokens (`packages/ui-tailwind/`)
3. Look at an existing similar component for patterns before writing a new one

## File Header — MANDATORY on every new file

```typescript
// Copyright 2025 TWIN Foundation
// SPDX-License-Identifier: Apache-2.0
```

ESLint enforces this. Missing headers will fail CI.

## New Component Checklist

```
packages/ui-components-react/src/<componentName>/
  <name>.tsx           ← main component
  <name>Props.ts       ← TypeScript interface with JSDoc
  <name>Colors.ts      ← color constants (if applicable)
  <name>Sizes.ts       ← size constants (if applicable)
  <name>.test.tsx      ← Vitest unit tests
  index.ts             ← re-exports
```

Then:
- Add to `packages/ui-components-react/src/index.tsx` barrel
- Add story to `apps/ui-storybook-react/`

## Props Pattern

```typescript
// <name>Props.ts
import type { FlowbiteXxxProps } from 'flowbite-react'

export type ComponentColor = 'primary' | 'secondary' | 'error' | 'warning' | 'success' | 'info' | 'plain' | 'ghost' | 'dark'
export type ComponentSize = 'xs' | 'sm' | 'md' | 'lg' | 'xl'

export interface ComponentNameProps extends Omit<FlowbiteXxxProps, 'color' | 'size'> {
  /** JSDoc description for every prop */
  color?: ComponentColor
  /** Size of the component */
  size?: ComponentSize
  /** Icon displayed on the left */
  leftIcon?: React.FC
}
```

## Class Merging — ALWAYS use cn()

```typescript
import { cn } from '../lib/utils'

// Good
<div className={cn('bg-surface-button text-primary', disabled && 'opacity-50', className)}>

// Bad — never do this
<div className={`bg-surface-button ${disabled ? 'opacity-50' : ''}`}>
```

## Design Tokens — NEVER hardcode colors

```tsx
// Good — semantic tokens only
<button className="bg-surface-button hover:bg-surface-button-hover text-primary">

// Bad
<button className="bg-blue-500 text-white">
```

Available token categories: `surface-*`, `text-*`, `success`, `warning`, `error`, `information`

## Code Style (enforced by ESLint + Prettier)

- Tabs (not spaces), width 2
- Single quotes
- No trailing commas
- Max 120 chars per line
- No `console.*` statements
- camelCase variables, PascalCase components/types
- Sort imports: stdlib → third-party → workspace → local
- Arrow functions: `x => x` not `(x) => x`

## Svelte-specific Rules

- Use Svelte 5 runes: `$props()`, `$state()`, `$derived()`
- Locale strings go in `locales/en.json` — run `npm run merge-locales` after
- Import from `flowbite-svelte` for base components

## Build Verification (run before every commit)

```bash
npm run build   # TypeScript compilation — must be clean
npm run test    # All tests must pass
npm run lint    # ESLint + Prettier + markdownlint + cspell — zero errors
```

## Commit Format

```
feat(button): add ghost color variant
fix(modal): correct z-index stacking
chore(tailwind): update surface token names
docs(readme): add usage examples
```

Types: `feat | fix | chore | docs | refactor | test | style`
Scope = package or component name.

## What NOT to Do

- No global state (Redux, Zustand, Context) — components are purely presentational
- No data-fetching inside components
- No direct Flowbite internal imports — extend via props
- No hardcoded colors
- No `--no-verify` on commits
