# TWIN UI — Product Design & UI Patterns

You are making visual or layout changes to the TWIN Foundation UI component library. This skill covers design tokens, component variants, and visual consistency rules.

## Design Token System

### Source of Truth
- **Figma** → CSS custom properties in `packages/ui-tailwind/src/css/`
- **`TailwindConfig` class** in `packages/ui-tailwind/src/tailwindConfig.ts` generates the Tailwind theme
- Never invent token names — only use what's defined in the css/ directory

### Token Categories

| Category | Examples |
|----------|---------|
| Surface | `surface-button`, `surface-button-hover`, `surface-button-alt` |
| Text | `text-primary`, `text-secondary`, `text-disabled`, `text-inverse` |
| System | `success`, `warning`, `error`, `information` |
| Brand | Defined in Figma brand collection — verify names in css/ files |

### Dark Mode
- Strategy: `darkMode: "class"` on root element
- Use `dark:` Tailwind variants for dark-mode overrides
- Tokens handle most theming automatically — prefer token classes over `dark:` overrides

### Typography
- Primary font: **Inter** (`font-sans`)
- No custom font sizes unless defined in Tailwind config

## Component Color Variants

All interactive components support these colors:
```
primary | secondary | error | warning | success | info | plain | ghost | dark
```

Mapping:
- `primary` — brand action, main CTA
- `secondary` — secondary action
- `error` — destructive, danger states
- `warning` — caution, confirmation states
- `success` — positive feedback
- `info` — informational
- `plain` — neutral/minimal styling
- `ghost` — transparent background, border-only
- `dark` — inverted/dark variant

## Size Scale

```
xs → sm → md → lg → xl
```

`md` is the default. Components should render correctly at all sizes.

## Icon Usage

Icons from `lucide-react` (70+ available):
```tsx
import { StarIcon } from '@twin.org/ui-components-react/icons/star'

// Left icon
<Button leftIcon={StarIcon}>Label</Button>

// Right icon
<Button rightIcon={ChevronRightIcon}>Next</Button>

// Icon only (no label)
<Button icon={TrashIcon} />
```

## Class Merging Pattern

```tsx
import { cn } from '../lib/utils'

// Compose classes safely
<div className={cn(
  'base-token-class',
  color === 'primary' && 'bg-surface-button text-inverse',
  disabled && 'opacity-50 cursor-not-allowed',
  className  // always pass through consumer className last
)}>
```

## Flowbite Component Extension

Components extend Flowbite for accessibility and base behavior. Override styles via:
1. Custom Tailwind classes via `className` prop (merged via `cn()`)
2. Flowbite `theme` prop for deep theme overrides
3. Never directly modify Flowbite source

## Storybook Documentation

Every visual change needs Storybook coverage:
```
apps/ui-storybook-react/src/<name>/<Name>.stories.tsx
```

Story requirements:
- `Default` story with baseline props
- One story per color variant (or use `argTypes` controls)
- One story per size variant
- `Disabled` state story
- `WithIcon` story if component supports icons

## Visual Consistency Rules

1. **No hardcoded colors** — always use semantic tokens
2. **Spacing** — use Tailwind spacing scale (`p-2`, `gap-4`, etc.)
3. **Borders** — use `rounded-*` with consistent radius (check similar components)
4. **Focus states** — must be visible for accessibility (`focus:ring-*`)
5. **Transitions** — use `transition-*` utilities for interactive states
6. **Hover/Active** — always define hover state for interactive elements

## Prettier Formatting for JSX

```tsx
// Max 100 chars per line — break long props
<Button
  color="primary"
  size="lg"
  leftIcon={StarIcon}
  onClick={handleClick}
  className="w-full"
>
  Label Text
</Button>
```

## Accessibility Requirements

- Buttons: use semantic `<button>` element (via Flowbite)
- Icons: decorative icons have `aria-hidden="true"`
- Icon-only buttons: must have `aria-label`
- Form inputs: must have associated `<label>`
- Color: never convey meaning through color alone (pair with icon or text)
