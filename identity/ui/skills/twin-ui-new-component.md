---
name: twin-ui-new-component
description: End-to-end workflow for creating a new React component in the TWIN UI monorepo — file structure, props pattern, design tokens, barrel export, tests, and Storybook story. Use when asked to add a new component.
---

# TWIN UI — Create New React Component

End-to-end guide for adding a new component to `packages/ui-components-react/`.

## Step 0 — Before You Write Anything

1. Read an existing similar component to understand patterns (e.g. Button for interactive, Badge for display-only)
2. Identify the closest Flowbite base component to extend
3. Confirm the component name (PascalCase): e.g. `TagInput`

Component directory: `packages/ui-components-react/src/<componentName>/`

---

## Step 1 — File Structure

Create all 6 files. Every file needs the Apache header:

```typescript
// Copyright 2024 IOTA Stiftung.
// SPDX-License-Identifier: Apache-2.0.
```

### `<name>Colors.ts` (if component has color variants)

```typescript
// Copyright 2024 IOTA Stiftung.
// SPDX-License-Identifier: Apache-2.0.
import {
	DARK,
	ERROR,
	GHOST,
	INFO,
	PLAIN,
	PRIMARY,
	SECONDARY,
	SUCCESS,
	WARNING
} from "../constants/colors";

export const ComponentColors = {
	Primary: PRIMARY,
	Secondary: SECONDARY,
	Error: ERROR,
	Warning: WARNING,
	Success: SUCCESS,
	Info: INFO,
	Plain: PLAIN,
	Ghost: GHOST,
	Dark: DARK
} as const;

export type ComponentColor = (typeof ComponentColors)[keyof typeof ComponentColors];
```

### `<name>Sizes.ts` (if component has size variants)

```typescript
// Copyright 2024 IOTA Stiftung.
// SPDX-License-Identifier: Apache-2.0.
import { EXTRA_LARGE, EXTRA_SMALL, LARGE, MEDIUM, SMALL } from "../constants/sizes";

export const ComponentSizes = {
	ExtraSmall: EXTRA_SMALL,
	Small: SMALL,
	Medium: MEDIUM,
	Large: LARGE,
	ExtraLarge: EXTRA_LARGE
} as const;

export type ComponentSize = (typeof ComponentSizes)[keyof typeof ComponentSizes];
```

### `<name>Props.ts`

```typescript
// Copyright 2024 IOTA Stiftung.
// SPDX-License-Identifier: Apache-2.0.
import type { ComponentPropsWithoutRef } from "react";
// Or extend Flowbite: import type { FlowbiteXxxProps } from "flowbite-react";

import type { ComponentColor } from "./componentColors";
import type { ComponentSize } from "./componentSizes";

export interface ComponentNameProps extends ComponentPropsWithoutRef<"div"> {
	/**
	 * The color variant of the component.
	 * @default ComponentColors.Primary
	 */
	color?: ComponentColor;

	/**
	 * The size of the component.
	 * @default ComponentSizes.Medium
	 */
	size?: ComponentSize;

	/**
	 * Additional CSS class names to apply.
	 */
	className?: string;
}
```

Rules:
- Extend Flowbite props when there's a direct base component — use `Omit<>` to replace `color`/`size`
- Full JSDoc `/** */` on every property
- Include `@default` tag when there's a sensible default

### `<name>.tsx`

```tsx
// Copyright 2024 IOTA Stiftung.
// SPDX-License-Identifier: Apache-2.0.
import { memo } from "react";

import { cn } from "../lib/utils";
import { ComponentColors } from "./componentColors";
import { ComponentSizes } from "./componentSizes";
import type { ComponentNameProps } from "./componentNameProps";

/**
 * ComponentName — short description.
 */
export const ComponentName = memo(function ComponentName({
	color = ComponentColors.Primary,
	size = ComponentSizes.Medium,
	className,
	children,
	...rest
}: ComponentNameProps) {
	const colorClasses: Record<string, string> = {
		[ComponentColors.Primary]: "bg-surface-button text-primary",
		[ComponentColors.Secondary]: "bg-surface-button-secondary text-secondary",
		[ComponentColors.Error]: "bg-error text-white",
		[ComponentColors.Warning]: "bg-warning text-white",
		[ComponentColors.Success]: "bg-success text-white",
		[ComponentColors.Info]: "bg-information text-white",
		[ComponentColors.Plain]: "bg-transparent text-primary border border-surface-button",
		[ComponentColors.Ghost]: "bg-transparent text-primary hover:bg-surface-button-hover",
		[ComponentColors.Dark]: "bg-surface-button-dark text-white"
	};

	const sizeClasses: Record<string, string> = {
		[ComponentSizes.ExtraSmall]: "text-xs px-2 py-1",
		[ComponentSizes.Small]: "text-sm px-3 py-1.5",
		[ComponentSizes.Medium]: "text-sm px-4 py-2",
		[ComponentSizes.Large]: "text-base px-5 py-2.5",
		[ComponentSizes.ExtraLarge]: "text-lg px-6 py-3"
	};

	return (
		<div className={cn(colorClasses[color], sizeClasses[size], className)} {...rest}>
			{children}
		</div>
	);
});

ComponentName.displayName = "ComponentName";
```

Rules:
- Use `memo()` — wrap in named function expression (not arrow) for displayName
- Set `displayName` explicitly at bottom
- Use semantic design tokens only — never hardcode colors
- Always spread `...rest` for extensibility
- Default props via destructuring, not `defaultProps`

### `index.ts`

```typescript
// Copyright 2024 IOTA Stiftung.
// SPDX-License-Identifier: Apache-2.0.
export { ComponentName } from "./componentName";
export { ComponentColors } from "./componentColors";
export { ComponentSizes } from "./componentSizes";
export type { ComponentNameProps } from "./componentNameProps";
export type { ComponentColor } from "./componentColors";
export type { ComponentSize } from "./componentSizes";
```

---

## Step 2 — Register in Barrel

Add to `packages/ui-components-react/src/index.tsx` in alphabetical order:

```typescript
// ComponentName
export { ComponentName } from "./componentName/componentName";
export { ComponentColors } from "./componentName/componentColors";
export { ComponentSizes } from "./componentName/componentSizes";
export type { ComponentNameProps } from "./componentName/componentNameProps";
export type { ComponentColor } from "./componentName/componentColors";
export type { ComponentSize } from "./componentName/componentSizes";
```

---

## Step 3 — Write Tests

See `twin-ui-qa` skill for full test guidance. Minimum tests:

- Renders with default props
- All color variants (9)
- All size variants (5)
- Disabled/error states if applicable
- Icon display if applicable
- onClick/onChange handlers if applicable
- Custom className is applied
- Snapshot test

---

## Step 4 — Write Storybook Story

See `twin-ui-storybook` skill for full story guidance.

---

## Step 5 — Verify

```bash
# From repo root
npm run build     # TypeScript must compile clean
npm run test      # All tests must pass
npm run lint      # Zero errors
```

Fix any TypeScript or lint errors before marking done.

---

## Design Token Reference

| Token | Usage |
|-------|-------|
| `bg-surface-button` | Primary button background |
| `bg-surface-button-hover` | Hover state |
| `bg-surface-button-secondary` | Secondary variant |
| `bg-surface-button-dark` | Dark variant |
| `text-primary` | Default text |
| `text-secondary` | Secondary/muted text |
| `bg-error` / `text-error` | Error state |
| `bg-warning` / `text-warning` | Warning state |
| `bg-success` / `text-success` | Success state |
| `bg-information` / `text-information` | Info state |

Always check `packages/ui-tailwind/src/tailwindConfig.ts` for the full token list.

---

## Common Mistakes to Avoid

- Hardcoding colors (`bg-blue-500`) — always use tokens
- String concatenation for classes — always use `cn()`
- Missing Apache header — ESLint will fail CI
- Importing from barrel in tests — import component file directly
- Missing JSDoc on props — keep all interface properties documented
- Forgetting `displayName` on memoized components
- Not adding to the `index.tsx` barrel
