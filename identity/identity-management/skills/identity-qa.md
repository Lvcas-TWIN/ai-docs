---
name: identity-qa
description: QA and testing standards for the identity-mvp project. Use when writing tests, reviewing test coverage, or verifying features work correctly.
---

# identity-mvp QA Guide

## Test Frameworks

| Layer | Framework | Command |
|---|---|---|
| Service integration tests | Vitest | `npm run test` (in `packages/identity-management-service`) |
| Frontend unit tests | Vitest | `npm run test:unit` |
| E2E | Playwright | `npm run test:e2e` |
| Coverage | Vitest v8 | `npm run test:coverage` or `npm run test:unit:coverage` |

## Service Package Tests (Vitest)

### Vitest Config — Critical Settings
```typescript
// packages/identity-management-service/vitest.config.ts
{
    testTimeout: 300000,  // 5 minutes — Argon2 is slow even at low params
    bail: 1,              // Stop on first failure
    pool: 'forks',        // No parallelism — tests can share file storage state
}
```

### ALWAYS use low Argon2 params in test setup
```typescript
const service = new IdentityManagementService({
    argon2TimeCost: 1,
    argon2MemoryCost: 1024,   // 1 MiB (production default is 64 MiB)
    argon2Parallelism: 1,
    passwordMinEntropy: 50,   // lower threshold for test passwords
    altchaHmacKey: 'test-hmac-key-for-tests-only',
    // ...wire MemoryEntityStorageConnector for each entity type
});
await service.start();
```

### Test File Locations
```
packages/identity-management-service/tests/
├── identityManagementService.spec.ts   ← Main service: login, onboarding, credentials, orgs
├── policyEnforcer.spec.ts              ← ABAC: role ranks, enforce, permissions, hierarchy validation
├── verificationProvider.spec.ts        ← KYC provider capabilities
├── verificationIntegration.spec.ts     ← Full KYC verification flow
├── kraOtpVerificationProvider.spec.ts  ← KRA OTP provider
├── onboardingReconciliation.spec.ts    ← Onboarding state reconciliation
├── auditUtils.spec.ts                  ← Audit logging utilities
├── entityCrud.spec.ts                  ← Entity CRUD test helper (test-only entity endpoint)
├── templateSchemaGenerator.spec.ts     ← JSON schema generation from templates
└── durationParser.spec.ts              ← ISO 8601 duration parsing
```

### Service Test Pattern
```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { MemoryEntityStorageConnector } from '@twin.org/entity-storage-connector-memory';
import { IdentityManagementService } from '../src/identityManagementService';

describe('IdentityManagementService', () => {
    let service: IdentityManagementService;

    beforeEach(async () => {
        const agentStorage = new MemoryEntityStorageConnector<AgentEntity>({ entitySchema: AgentEntity });
        // ... create connectors for each entity type

        service = new IdentityManagementService({
            argon2TimeCost: 1,
            argon2MemoryCost: 1024,
            argon2Parallelism: 1,
            passwordMinEntropy: 50,
            altchaHmacKey: 'test-key',
            agentEntityStorageConnector: agentStorage,
            // ...other connectors
        });
        await service.start();
    });

    it('should login with valid credentials', async () => {
        const result = await service.login('test@example.com', 'ValidP@ssw0rd123');
        expect(result.token).toBeDefined();
        expect(result.userId).toBeDefined();
    });

    it('should reject invalid password', async () => {
        await expect(service.login('test@example.com', 'wrong'))
            .rejects.toMatchObject({ properties: { code: 'agentNotFound' } });
    });
});
```

### PolicyEnforcer Test Pattern
```typescript
import { describe, it, expect } from 'vitest';
import { PolicyEnforcer } from '../src/policyEnforcer';

describe('PolicyEnforcer', () => {
    it('derives role ranks from default policy', async () => {
        const enforcer = new PolicyEnforcer(agentStorage);
        await enforcer.initialize();

        const ranks = enforcer.getRoleRank();
        expect(ranks['primary-owner']).toBeGreaterThan(ranks['owner']);
        expect(ranks['owner']).toBeGreaterThan(ranks['admin']);
        expect(ranks['admin']).toBeGreaterThan(ranks['member']);
    });

    it('grants template.create to admin, denies to member', async () => {
        const allowed = await enforcer.enforce(adminUserId, orgId, 'template', 'create');
        const denied  = await enforcer.enforce(memberUserId, orgId, 'template', 'create');
        expect(allowed).toBe(true);
        expect(denied).toBe(false);
    });

    it('rejects non-linear role hierarchy via policyPaths', async () => {
        const enforcer = new PolicyEnforcer(agentStorage, undefined, ['./tests/fixtures/branching-policy.csv']);
        await expect(enforcer.initialize()).rejects.toThrow('roleHierarchyNotLinear');
    });

    it('loads additional permissions from policyPaths', async () => {
        const enforcer = new PolicyEnforcer(agentStorage, undefined, ['./tests/fixtures/extra-policy.csv']);
        await enforcer.initialize();
        const allowed = await enforcer.enforce(userId, orgId, 'custom-resource', 'read');
        expect(allowed).toBe(true);
    });
});
```

## Frontend Unit Tests (Vitest)

### Location — Mirror source structure exactly
```
lib/utils/formatDate.ts          →  tests/unit/lib/utils/formatDate.test.ts
lib/schemas/auth.ts              →  tests/unit/lib/schemas/auth.test.ts
hooks/useMyHook.ts               →  tests/unit/hooks/useMyHook.test.ts
lib/utils/role.utils.ts          →  tests/unit/lib/utils/role.utils.test.ts
```

### Standard Unit Test Pattern
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

### What to Unit Test
- All `lib/utils/` functions — every branch
- All `lib/schemas/` Zod schemas — valid + invalid + edge cases
- Custom hooks with `renderHook` from `@testing-library/react`
- Role utilities (`maxRank`, `invitableRoles`, `assignableRoles`)
- Permission helpers

### Mocking
```typescript
import { vi } from 'vitest';
vi.mock('@/lib/database/database', () => ({
    database: {
        getPermissionsForUser: vi.fn(),
        getAgentsByOrganization: vi.fn(),
    }
}));
```

## E2E Tests (Playwright)

### Location
```
e2e/tests/           ← test specs (.spec.ts)
e2e/page-objects/    ← page object models
e2e/fixtures/        ← shared test fixtures
```

### Page Object Model — always use POM, never raw selectors in spec files
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
import { test } from '@playwright/test';
import { TeamPage } from '../page-objects/TeamPage';

test('admin can see team members', async ({ page }) => {
    const teamPage = new TeamPage(page);
    await teamPage.goto();
    await teamPage.expectMemberVisible('admin@example.com');
});
```

### Critical Paths to Cover
1. Login → profile redirect
2. Registration → full onboarding flow completion
3. Email verification (with token)
4. Add team member (invite flow)
5. Create credential template
6. Issue / approve credential request
7. Settings update
8. Role change (owner demotes admin)
9. Permission gate (member cannot access admin-only actions)

## Testing Permission Logic — Always Test Both Sides

```typescript
it('admin can create templates', async () => {
    // run as admin context
    const result = await service.createTemplate(orgId, validTemplateData);
    expect(result.template).toBeDefined();
});

it('member cannot create templates', async () => {
    // run as member context
    await expect(service.createTemplate(orgId, validTemplateData))
        .rejects.toMatchObject({ properties: { code: 'insufficientPermissions' } });
});
```

## Testing Role Change Rules

```typescript
it('admin cannot edit another admin (equal rank)', async () => {
    // Admin (rank 1) cannot modify admin (rank 1) — strict > required
    await expect(service.updateAgentRole(orgId, otherAdminId, 'member'))
        .rejects.toMatchObject({ properties: { code: 'roleChangeNotAllowed' } });
});

it('owner can demote admin', async () => {
    // Owner (rank 2) > admin (rank 1) — allowed
    const result = await service.updateAgentRole(orgId, adminId, 'member');
    expect(result.agent.memberOf[0].role).toBe('member');
});

it('cannot self-change role', async () => {
    await expect(service.updateAgentRole(orgId, selfId, 'member'))
        .rejects.toMatchObject({ properties: { code: 'cannotChangeOwnRole' } });
});
```

## Testing Onboarding State Machine

```typescript
it('cannot complete step if dependencies not met', async () => {
    // Try ORGANIZATION_DETAILS before PERSONAL_DETAILS completes
    await expect(service.updateOnboardingState(id, 'ORGANIZATION_DETAILS', data, status, false))
        .rejects.toMatchObject({ properties: { code: 'stepDependenciesNotMet' } });
});

it('cannot go back to a completed step', async () => {
    await expect(service.updateOnboardingState(id, 'EMAIL_VERIFICATION', data, completedStatus, false))
        .rejects.toMatchObject({ properties: { code: 'cannotGoBackToCompletedStep' } });
});
```

## data-testid Conventions

| Element | Convention |
|---|---|
| Form inputs | `data-testid="email-input"`, `data-testid="password-input"` |
| Submit / continue | `data-testid="continue-button"`, `data-testid="submit-button"` |
| Error messages | `data-testid="login-error"`, `data-testid="verification-error"` |
| Modal triggers | `data-testid="add-member-button"` |
| Table rows | `data-testid="table-row-{id}"` |
| Key sections | `data-testid="profile-content"`, `data-testid="members-table"` |

## CI Checks (Run on Every PR)

1. `npm run lint`
2. `npm run typecheck`
3. `npm run i18n-check`
4. `npm run test` (service package)
5. Playwright E2E on Vercel preview

## Before Marking a Feature Complete

- [ ] Unit tests written for all new utility functions and schemas
- [ ] Service tests cover the happy path
- [ ] Service tests cover permission denials (both sides)
- [ ] E2E test written for the user-facing happy path
- [ ] `npm run test` passes (service)
- [ ] `npm run typecheck` passes
- [ ] `npm run build` passes
- [ ] Manually tested on local dev server
- [ ] Error states tested (empty state, network error, invalid input)
- [ ] Permission gates tested (member cannot do admin-only actions)
- [ ] Role change edge cases tested (equal rank, self-change, primary-owner protection)
