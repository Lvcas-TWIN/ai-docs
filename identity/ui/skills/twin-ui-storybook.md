---
name: twin-ui-storybook
description: Guide for writing and updating Storybook stories in the TWIN UI monorepo. Use when creating new stories, adding story variants, or updating existing stories after component changes.
---

# TWIN UI — Storybook Stories

Guide for writing and updating Storybook stories for `apps/ui-storybook-react/`.

## Story File Location

```
apps/ui-storybook-react/src/stories/<ComponentName>.stories.tsx
```

Run Storybook: `cd apps/ui-storybook-react && npm run storybook` (port 6006)

---

## Story File Template

```tsx
// Copyright 2024 IOTA Stiftung.
// SPDX-License-Identifier: Apache-2.0.
import type { Meta, StoryObj } from "@storybook/react";
import { ComponentName, ComponentColors, ComponentSizes } from "@twin.org/ui-components-react";

const meta: Meta<typeof ComponentName> = {
	title: "Components/ComponentName",
	component: ComponentName,
	parameters: {
		layout: "centered"
	},
	tags: ["autodocs"],
	argTypes: {
		color: {
			control: "select",
			options: Object.values(ComponentColors),
			description: "The color variant"
		},
		size: {
			control: "select",
			options: Object.values(ComponentSizes),
			description: "The size variant"
		}
	}
};

export default meta;
type Story = StoryObj<typeof ComponentName>;

// ── Default ──────────────────────────────────────────────────────────
export const Default: Story = {
	args: {
		color: ComponentColors.Primary,
		size: ComponentSizes.Medium
	}
};

// ── Color Variants ────────────────────────────────────────────────────
export const Primary: Story = {
	args: { color: ComponentColors.Primary }
};

export const Secondary: Story = {
	args: { color: ComponentColors.Secondary }
};

export const Error: Story = {
	args: { color: ComponentColors.Error }
};

export const Warning: Story = {
	args: { color: ComponentColors.Warning }
};

export const Success: Story = {
	args: { color: ComponentColors.Success }
};

export const Info: Story = {
	args: { color: ComponentColors.Info }
};

export const Ghost: Story = {
	args: { color: ComponentColors.Ghost }
};

export const Plain: Story = {
	args: { color: ComponentColors.Plain }
};

export const Dark: Story = {
	args: { color: ComponentColors.Dark }
};

// ── Size Variants ─────────────────────────────────────────────────────
export const Sizes: Story = {
	render: () => (
		<div className="flex items-center gap-4 flex-wrap">
			{Object.values(ComponentSizes).map(size => (
				<ComponentName key={size} size={size}>
					{size}
				</ComponentName>
			))}
		</div>
	)
};

// ── All Colors Grid ───────────────────────────────────────────────────
export const AllColors: Story = {
	render: () => (
		<div className="flex flex-wrap gap-3">
			{Object.values(ComponentColors).map(color => (
				<ComponentName key={color} color={color}>
					{color}
				</ComponentName>
			))}
		</div>
	)
};
```

---

## Meta Configuration Rules

| Field | Value |
|-------|-------|
| `title` | `"Components/ComponentName"` — matches sidebar hierarchy |
| `layout` | `"centered"` (default) or `"fullscreen"` for page-level components |
| `tags` | Always include `["autodocs"]` — generates the Props table |
| `argTypes` | Define for all main props; use `control: "select"` for enum-like props |

---

## Required Stories Per Component

Every component story file must cover:

1. **`Default`** — shows the component with sensible defaults
2. **All color variants** — one story per color (Primary, Secondary, Error, Warning, Success, Info, Ghost, Plain, Dark)
3. **Size variants** — render all sizes together in one story for visual comparison
4. **`AllColors`** — a grid/flex row showing all colors at once (good for quick review)
5. **Interactive states** — if component has onClick/onChange, show a story with a handler
6. **Icon variants** — if component supports icons (leftIcon, rightIcon, icon-only)
7. **Disabled state** — if applicable
8. **Edge cases** — long text, empty, loading state, etc.

---

## Adding Icons to Stories

```tsx
import { HiCheck, HiX } from "react-icons/hi";

export const WithIcons: Story = {
	args: {
		leftIcon: HiCheck,
		rightIcon: HiX
	}
};

export const IconOnly: Story = {
	args: {
		icon: HiCheck,
		iconOnly: true
	}
};
```

---

## Interactive Stories (with handlers)

```tsx
export const WithOnClick: Story = {
	args: {
		onClick: () => alert("Clicked!")
	}
};

// Or use Storybook actions:
import { fn } from "@storybook/test";

export const Clickable: Story = {
	args: {
		onClick: fn()
	}
};
```

---

## Render Function Pattern (for composite/grid stories)

Use `render:` when you need multiple instances:

```tsx
export const ColorGrid: Story = {
	render: () => (
		<div className="grid grid-cols-3 gap-4 p-4">
			<ComponentName color="primary">Primary</ComponentName>
			<ComponentName color="secondary">Secondary</ComponentName>
			<ComponentName color="error">Error</ComponentName>
		</div>
	)
};
```

---

## Updating an Existing Story

When a component gains new props:
1. Add new `argTypes` entry in meta
2. Add new story(ies) demonstrating the prop
3. Update `Default` story args if the default changed
4. Verify `AllColors`/`Sizes` stories still render correctly

When a component is renamed or props renamed:
1. Update imports at top of story file
2. Update all `args` references
3. Run Storybook to verify no errors

---

## Common Mistakes

- Missing `tags: ["autodocs"]` — no auto-generated Props table
- Importing from component file path instead of `@twin.org/ui-components-react` package
- Not showing all 9 color variants — reviewers need to see the full palette
- Using inline style (`style={{ color: "blue" }}`) — always use Tailwind classes
- Missing Apache header on the story file
- Story title doesn't match the sidebar structure (`"Components/ComponentName"`)
