---
name: identity-qa
description: QA and testing standards for the identity-mvp project. Use when writing tests, reviewing test coverage, or verifying features work correctly.
---

# identity-mvp QA Guide

## Test Frameworks
- **Unit:** Vitest (`npm run test:unit`)
- **E2E:** Playwright (`npm run test:e2e`)
- **Coverage:** `npm run test:unit:coverage`

## Unit Tests (Vitest)

### Location
`tests/unit/` — mirror the source structure exactly.
```
lib/utils/formatDate.ts  →  tests/unit/lib/utils/formatDate.test.ts
lib/schemas/auth.ts      →  tests/unit/lib/schemas/auth.test.ts
hooks/useMyHook.ts       →  tests/unit/hooks/useMyHook.test.ts
```

### Pattern
```typescript
import { describe, it, expect, vi } from 'vitest';
import { myFunction } from '@/lib/utils/myFunction';

describe('myFunction', () => {
    it('returns expected value for valid input', () => {
        expect(myFunction('valid')).toBe('expected');
    });

    it('handles empty input', () => {
        expect(myFunction('')).toBe('fallback');
    });

    it('throws on invalid input', () => {
        expect(() => myFunction(null as any)).toThrow();
    });
});
```

### What to unit test
- All `lib/utils/` functions — test every branch
- All `lib/schemas/` Zod schemas — valid + invalid + edge cases
- Custom hooks with `renderHook` from `@testing-library/react`
- Pure business logic (role utils, permission helpers, onboarding utils)

### Mocking
```typescript
import { vi } from 'vitest';
vi.mock('@/lib/database/database', () => ({ database: { getPermissionsForUser: vi.fn() } }));
```

## E2E Tests (Playwright)

### Location
```
e2e/tests/          ← test specs
e2e/page-objects/   ← page object models
e2e/fixtures/       ← shared fixtures
```

### Page Object Model — always
```typescript
// e2e/page-objects/TeamPage.ts
import type { Page } from '@playwright/test';

export class TeamPage {
    constructor(private page: Page) {}

    async goto() { await this.page.goto('/en/team'); }
    async clickAddMember() { await this.page.getByTestId('add-member-button').click(); }
    async expectMemberVisible(email: string) {
        await expect(this.page.getByText(email)).toBeVisible();
    }
}

// e2e/tests/team.spec.ts
import { test, expect } from '@playwright/test';
import { TeamPage } from '../page-objects/TeamPage';

test.describe('Team management', () => {
    test('admin can see team members', async ({ page }) => {
        const teamPage = new TeamPage(page);
        await teamPage.goto();
        await teamPage.expectMemberVisible('admin@example.com');
    });
});
```

### Critical paths to cover with E2E
1. Login → profile redirect
2. Registration → onboarding flow completion
3. Email verification (with token)
4. Add team member (invite)
5. Create credential template
6. Issue/approve credential request
7. Settings update

## data-testid Conventions

Add to all interactive elements and key content:

| Element | Convention |
|---|---|
| Form inputs | `data-testid="email-input"`, `data-testid="password-input"` |
| Submit buttons | `data-testid="continue-button"`, `data-testid="submit-button"` |
| Error messages | `data-testid="login-error"`, `data-testid="verification-error"` |
| Modal triggers | `data-testid="add-member-button"` |
| Table rows | `data-testid="table-row-{id}"` |
| Key sections | `data-testid="profile-content"`, `data-testid="members-table"` |

## Before Marking a Feature Complete
- [ ] Unit tests written for all new utility functions and schemas
- [ ] E2E test written for the happy path
- [ ] `npm run test:unit` passes
- [ ] `npm run typecheck` passes
- [ ] `npm run build` passes
- [ ] Manually tested on dev server
- [ ] Error states tested (empty state, network error, invalid input)
- [ ] Permissions tested (what happens if user lacks permission)

## CI Checks (run on every PR)
1. `npm run lint`
2. `npm run typecheck`
3. `npm run i18n-check`
4. Playwright E2E on Vercel preview (`playwright-vercel-preview.yml`)
