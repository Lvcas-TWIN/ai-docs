# TWIN UI — QA & Testing Standards

You are reviewing or writing tests for the TWIN Foundation UI component library (React 19 + Svelte 5).

## Test Stack

- **Vitest** 4.0+ with jsdom environment
- **@testing-library/react** 16 — render, screen, fireEvent, userEvent
- **@testing-library/jest-dom** — DOM matchers (toBeInTheDocument, toHaveClass, etc.)

## File Placement

Test lives alongside the component:
```
src/button/button.tsx
src/button/button.test.tsx   ← test here
src/button/__snapshots__/    ← auto-generated snapshots
```

## Required Apache Header

```typescript
// Copyright 2025 TWIN Foundation
// SPDX-License-Identifier: Apache-2.0
```

## Test File Template

```tsx
// Copyright 2025 TWIN Foundation
// SPDX-License-Identifier: Apache-2.0

import { render, screen, fireEvent } from '@testing-library/react'
import { describe, expect, it, vi } from 'vitest'

// IMPORTANT: Import from the component file, NOT from index.ts
import { ComponentName } from './componentName'

describe('ComponentName', () => {
  it('renders with default props', () => {
    render(<ComponentName />)
    expect(screen.getByRole('...')).toBeInTheDocument()
  })

  it('applies color variant class', () => {
    const { container } = render(<ComponentName color="primary" />)
    expect(container.firstChild).toHaveClass('...')
  })

  it('fires onClick handler', () => {
    const handler = vi.fn()
    render(<ComponentName onClick={handler}>Click</ComponentName>)
    fireEvent.click(screen.getByRole('button'))
    expect(handler).toHaveBeenCalledOnce()
  })

  it('matches snapshot', () => {
    const { container } = render(<ComponentName />)
    expect(container).toMatchSnapshot()
  })
})
```

## Coverage Checklist Per Component

| Test | Description |
|------|-------------|
| Default render | Renders without props/errors |
| All color variants | `primary`, `secondary`, `error`, `warning`, `success`, `info`, `plain`, `ghost`, `dark` |
| All size variants | `xs`, `sm`, `md`, `lg`, `xl` |
| Disabled state | Renders disabled, click doesn't fire |
| Icon props | `leftIcon`, `rightIcon`, `icon` (icon-only mode) |
| User interaction | click, change, focus/blur as applicable |
| Custom className | Passed className merges correctly via `cn()` |
| Snapshot | Baseline snapshot for visual regression |

## Query Priority (accessibility-first)

1. `getByRole` — preferred (button, textbox, checkbox, etc.)
2. `getByLabelText` — for form fields
3. `getByText` — for content assertions
4. `getByTestId` — last resort only

## Running Tests

```bash
# Root level
npm run test                          # All packages

# React package
cd packages/ui-components-react
npm run test                          # Run once
npm run test:coverage                 # With coverage
npx vitest --update-snapshots        # Refresh snapshots
```

## Coverage Targets

- Include: `src/**/*.ts`, `src/**/*.tsx`
- Exclude: `src/index.ts`, `src/**/models/**`
- Reporters: text (console) + lcov (for CI)

## CI Failure Checklist

If tests fail in CI:
1. Run `npm run test` locally — replicate the failure
2. Check `packages/ui-components-react/vitest.config.ts` for jsdom/setup issues
3. Check that test imports from component file (not barrel)
4. Check snapshot is committed and not stale (`--update-snapshots`)
5. Check file header is present (Apache-2.0)

## Common Mistakes

- Importing from `index.ts` barrel in tests — import the component file directly
- Missing file header — ESLint will fail
- Not updating snapshots after intentional UI change
- Using `querySelector` instead of Testing Library queries
- Testing implementation details instead of user-observable behavior
