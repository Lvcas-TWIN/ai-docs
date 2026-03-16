---
name: identity-business-logic
description: Business logic, ABAC permissions, role hierarchy, credential flows, and core domain rules for the identity-mvp project. Use when implementing or debugging permission checks, role management, credential workflows, or any domain-specific behavior.
---

# identity-mvp Business Logic Guide

## ABAC Permission System

Permissions are fetched server-side per request and passed as props to client components. They are resource+action boolean maps.

### Types
```typescript
// In hooks/usePermissions.ts
export type PermissionMap = Record<string, Record<string, boolean>>;
// { 'template': { 'list': true, 'create': false }, ... }

export type RoleRankMap = Record<string, number>;
// { 'admin': 3, 'manager': 2, 'member': 1 }
// Higher number = higher privilege
```

### Server-Side Fetch (page.tsx)
```typescript
import { database } from '@/lib/database/database';
import type { PermissionMap } from '@/hooks/usePermissions';

let permissions: PermissionMap = {};
try {
    const permResult = await database.getPermissionsForUser(organization.id);
    permissions = permResult.permissions;
} catch (error) {
    logError(error, 'PageName: Error fetching permissions');
}

// Pass boolean props to client component
const canCreate = permissions?.template?.create ?? false;
const canEdit = permissions?.['agent-role']?.update ?? false;
```

### Client-Side Hook
```typescript
import { usePermissions } from '@/hooks/usePermissions';

const { permissions, roleRank, hasPermission, isLoading } = usePermissions({
    organizationId,
    initialData,  // pass SSR data to avoid loading flash
});

hasPermission('template', 'create')    // boolean
hasPermission('agent-role', 'update')  // boolean
```

### Known Permission Resources & Actions
| Resource | Actions |
|---|---|
| `template` | `list`, `create`, `edit`, `delete` |
| `credential-request` | `approve`, `reject`, `view` |
| `invitation` | `create` |
| `agent` | `remove` |
| `agent-role` | `update` |
| `organization` | `settings`, `delete` |

## Role Hierarchy

### Rules — NEVER violate
- **Never hardcode role names** (`'admin'`, `'member'`) in logic — role names come from the backend
- Use `roleRank[userRole] ?? 0` for numeric comparison
- Higher rank number = more permissions
- `isPrivilegedRole = (roleRank[userRole] ?? 0) > 0`

### Role Utility Functions (`lib/utils/role.utils.ts`)
```typescript
// Highest privilege level in the org
maxRank(roleRank: RoleRankMap): number

// Roles this user can invite (rank ≤ own AND < max)
invitableRoles(roleRank: RoleRankMap, currentUserRole: string): string[]

// Roles this user can assign to others (rank < own AND < max)
// Cannot promote to equal or above yourself
assignableRoles(roleRank: RoleRankMap, currentUserRole: string): string[]
```

### Guarding Role Changes
```typescript
// Can the current user edit target user's role?
const currentUserRank = roleRank[currentUserRole] ?? 0;
const targetUserRank = roleRank[targetUserRole] ?? 0;
const canEdit = currentUserRank > targetUserRank;  // strictly greater
```

## Organization & Agent Domain

### Organization Role Types (`lib/interfaces/IOrganizationRoleName.ts`)
```typescript
export type IOrganizationRoleName = 'issuer' | 'holder' | 'issuer-holder';
```
- `issuer` — can create templates and issue credentials
- `holder` — can receive and present credentials
- `issuer-holder` — both

### Database Queries (server-side only)
```typescript
// Get agents in organization
const { agents, cursor } = await database.getAgentsByOrganization(organizationId);

// Get permissions for current user
const { permissions } = await database.getPermissionsForUser(organizationId);
```

## Credential Workflow

### Template → Issuance → Approval Flow
1. Privileged user creates a **template** (schema for a credential)
2. Agent submits a **credential request** against a template
3. Privileged user **approves** or **rejects** the request
4. Approved request becomes an issued credential

### Permission Gates
| Action | Permission |
|---|---|
| Create template | `template.create` |
| Edit template | `template.edit` |
| Delete template | `template.delete` |
| Approve/reject request | `credential-request.approve` / `.reject` |
| View requests | `credential-request.view` |

## Onboarding Business Rules

### Account Type Choices
```typescript
type AccountType = 'issuer' | 'holder' | 'issuer-holder';
```
- Determines which credential operations the organization can perform
- Set once during onboarding; affects feature availability after

### Required vs Optional Steps
- **Required:** EMAIL_VERIFICATION, EMAIL_CONFIRMATION, PERSONAL_DETAILS, INTRODUCTION, ACCOUNT_TYPE, ORGANIZATION_DETAILS, ORGANIZATION_DETAILS_EXTRA, COMPLETED
- **Optional:** KYC_KRA_KENYA (only present if enabled by backend for Kenya-registered orgs)

### Organization Identifiers
| Field | Purpose |
|---|---|
| LEI | Legal Entity Identifier (global) |
| EORI | Economic Operators Registration and Identification (EU trade) |
| BRN | Business Registration Number (Kenya KRA) |

## Authentication & Session

### Getting the Authenticated Agent (server-side)
```typescript
import { getAuthenticatedAgent } from '@/lib/auth/getAuthenticatedAgent';

const { agent, organization } = await getAuthenticatedAgent();
// Throws/redirects on unauthenticated
```

## Forms & Validation

All forms use react-hook-form + Zod. Schemas live in `lib/schemas/`.

### Key Schemas
| File | Validates |
|---|---|
| `onboarding.schemas.ts` | Personal details, org details, org details extra |
| `verification.schemas.ts` | Business verification (BRN, KRA PIN), OTP |
| `registration.schemas.ts` | User registration (email, password) |
| `auth.schemas.ts` | Login credentials |
| `template.schemas.ts` | Credential template creation |

### Telephone Validation
Uses `telephoneRegex` from `lib/constants/`. Do NOT write custom phone regex — import from there.

## API Calls (Client Components)

```typescript
import { apiFetch } from '@/lib/utils/api-fetch';

// Always use apiFetch — it handles auth headers and error normalization
const result = await apiFetch('/api/some-endpoint', {
    method: 'POST',
    body: JSON.stringify(data),
});
```

## Error Handling

```typescript
import { logError } from '@/lib/utils/error-logging';

try {
    // ...
} catch (error) {
    logError(error, 'ComponentName: descriptive context');
}
```

## Data Fetching Patterns

| Context | Method |
|---|---|
| Server component / page.tsx | `database.*` directly |
| Client component | TanStack Query hook in `hooks/` |
| API call from client | `apiFetch` from `@/lib/utils/api-fetch` |

### TanStack Query Hook Pattern
```typescript
// hooks/useMyData.ts
import { useQuery } from '@tanstack/react-query';
import { apiFetch } from '@/lib/utils/api-fetch';

export function useMyData(organizationId: string, initialData?: MyDataType) {
    return useQuery({
        queryKey: ['myData', organizationId],
        queryFn: () => apiFetch<MyDataType>(`/api/my-data?orgId=${organizationId}`),
        initialData,
        staleTime: 5 * 60 * 1000,
    });
}
```

## TypeScript Rules

- Never use `any` — use `unknown` and narrow with guards
- Use type imports: `import { type MyType } from '...'`
- All interface files in `lib/interfaces/` follow `I` prefix convention: `IAgent`, `IOrganization`

## Checklist Before Submitting Business Logic Changes

- [ ] No hardcoded role names in comparisons
- [ ] Permission checks use `?? false` fallback
- [ ] Role editing guards use strict `>` (not `>=`) comparison
- [ ] Server-side data access uses `database.*` only
- [ ] Client-side mutations use `apiFetch` via a TanStack Query hook
- [ ] Errors caught and logged with `logError`
- [ ] Zod schema used for any user input validation
