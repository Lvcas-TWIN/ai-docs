# TWIN UI — Project Rules for Claude

## Tech Stack

- **Monorepo** with npm workspaces (Node.js >=20)
- **TypeScript** 5.9+ — strict mode, all files require Apache-2.0 file header
- **React** 19 + **Svelte** 5 — dual-framework component library
- **Tailwind CSS** 3 + **Flowbite** — base styling; custom design tokens via `@twin.org/ui-tailwind`
- **Rollup** for ESM/CJS bundles; **Vite** for Storybook; **Vitest** for tests
- **Storybook** 8 for component docs (React: port 6006; Svelte: separate app)

## Workspace Layout

```
packages/
  ui-tailwind/              # Design tokens — Tailwind theme generator
  ui-components-react/      # React components (47+)
  ui-components-svelte/     # Svelte components
apps/
  ui-storybook-react/       # React Storybook
  ui-storybook-svelte/      # Svelte Storybook
scripts/                    # Build automation
```

## Component Authoring Rules

### File structure — every component must have:
```
<name>/
  <name>.tsx (or .svelte)   # Main component
  <name>Props.ts            # TypeScript interface
  <name>Colors.ts           # Color constants (if applicable)
  <name>Sizes.ts            # Size constants (if applicable)
  <name>.test.tsx           # Unit tests (required)
  index.ts                  # Clean re-export
```

### Props pattern
- Extend Flowbite component props where applicable
- Color variants: `primary | secondary | error | warning | success | info | plain | ghost | dark`
- Size variants: `xs | sm | md | lg | xl`
- Add icons via `leftIcon`, `rightIcon`, `icon` (icon-only)
- Full JSDoc on every exported interface property

### CSS merging
- Always use `cn()` from `src/lib/utils.ts` to combine Tailwind classes
- `cn()` wraps `clsx` + `tailwind-merge` — never concatenate class strings manually

### Design tokens
- Use semantic token names only: `surface-button`, `surface-button-hover`, `text-primary`, etc.
- Never hardcode hex/rgb colors — always reference Tailwind token classes
- Dark mode via `darkMode: "class"` — use `dark:` variants appropriately
- Import theme from `@twin.org/ui-tailwind`

## Code Quality Rules

- **No `console` statements** — ESLint enforces this
- **camelCase** for variables and functions; **PascalCase** for components/types
- **Max 120 chars** per line
- **Imports** must be sorted (`eslint-plugin-simple-import-sort`)
- **Apache-2.0 file header** required on every source file
- **No trailing commas**, **single quotes**, **tabs** for indentation (see `.prettierrc`)
- Run `npm run format` before committing

## i18n

- Svelte components use JSON locale files in `locales/` — add keys when adding user-visible strings
- Run `npm run merge-locales` (in svelte package) after adding locale keys
- React package: no i18n setup yet — pure props for labels

## Testing Rules

- Test file lives alongside source: `<name>.test.tsx`
- Use `@testing-library/react` + `vitest`
- Test: rendering, prop variations, user interactions, icon display
- Coverage reporters: text + lcov
- **Never** import from `index.ts` barrel in tests — import the component file directly
- Run `npm run test:coverage` to verify before submitting

## Build Verification (MANDATORY before push)

```bash
npm run build          # TypeScript compilation — must be clean
npm run test           # All tests must pass
npm run lint           # ESLint + Prettier + markdownlint + cspell — zero errors
```

- **Never skip** these three — CI enforces all of them
- Build outputs: ESM (`.mjs`), CJS (`.cjs`), Types (`.d.ts`), CSS, Docs

## Commit Rules

- **Conventional Commits** enforced via commitlint + husky
- Format: `type(scope): message` — e.g., `feat(button): add ghost color variant`
- Valid types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `style`
- Scope = package or component name
- **No "Co-Authored-By: Claude"** or similar attribution
- Ask for explicit confirmation before `git push`, `git merge`, `git rebase`, or any force/destructive op

## Storybook

- Every new React component needs a `.stories.tsx` file in the storybook app
- Stories should cover all major prop variants (colors, sizes, states)
- Run Storybook: `cd apps/ui-storybook-react && npm run storybook`

## Adding a New Component — Checklist

- [ ] Create directory with all required files (see structure above)
- [ ] Export from `packages/ui-components-react/src/index.tsx` barrel
- [ ] Add Storybook story in `apps/ui-storybook-react/`
- [ ] Write unit tests
- [ ] Add Apache-2.0 file header to every new file
- [ ] Run `npm run build && npm run test && npm run lint` — all green
- [ ] Add locale key if component has any user-visible string (Svelte only, for now)

## Do NOT

- Do not hardcode colors — use design tokens only
- Do not add global state management — components are stateless/presentational
- Do not add data-fetching logic inside components
- Do not import Flowbite internals directly — extend via props pattern
- Do not skip the file header on new files (ESLint will catch it, but don't cause the CI failure)
