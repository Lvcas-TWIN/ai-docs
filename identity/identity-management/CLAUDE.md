# TWIN Identity Management — Project Rules

## Project Structure

```
identity-management/
├── apps/
│   └── identity-management-node/       ← REST API server (port 3001 in dev)
└── packages/
    ├── identity-management-service/    ← Core business logic + Casbin RBAC
    ├── identity-management-models/     ← Shared TypeScript interfaces
    └── identity-management-rest-client/ ← Generated REST client
```

The node app is a thin plugin wrapper. All business logic lives in the service package.

## Skills — Load These Before Working on Related Tasks

| Task | Skill |
|---|---|
| Writing/editing any code | `/identity-development` |
| Permissions, roles, business logic | `/identity-business-logic` |
| Writing or reviewing tests | `/identity-qa` |
| Onboarding flow, steps, state machine | `/identity-onboarding` |
| Integrating a new external team/service | `/identity-new-service` |
| UI/visual changes | `/identity-product-design` |

## Build Commands

```bash
npm run dist:no-test   # build without tests (fastest, use for iteration)
npm run dist           # full build + tests
npm run test           # vitest only (5-min timeout, bail:1, no parallelism)
npm run dev            # start dev server with watch (port 3001)
```

## Critical Rules

### Never violate these
- **Never hardcode role names** in logic — always use `rankOf()` for rank comparison
- **Always `await this.ensureEnforcerReady()`** before any `policyEnforcer` call
- **Never mutate stored entities directly** — assign filtered result to a new variable
- **Role change guard:** `actorRank > targetRank` (strict) AND `actorRank >= newRank`
- **policyPaths** for policy extensions — never modify `default-policy.csv`
- Role hierarchy MUST remain a single linear chain (no branching)

### Storage
- `FileEntityStorageConnector` loads from disk on startup, keeps in-memory, writes on mutation
- **Stop the server before editing any `.local-data/*/store.json` file manually**

### TypeScript
- Never use `any` — use `unknown` and narrow with type guards
- Interface naming: `I` prefix (`IAgent`, `IOrganization`)
- An agent has ONE role per org — `IAgentMemberOf.role` is a string, not an array

## Key Files

| File | Purpose |
|---|---|
| `packages/identity-management-service/src/identityManagementService.ts` | Core service (~52 public methods) |
| `packages/identity-management-service/src/identityManagementRoutes.ts` | REST route definitions |
| `packages/identity-management-service/src/policyEnforcer.ts` | Casbin wrapper |
| `packages/identity-management-service/src/policies/default-policy.csv` | RBAC rules |
| `packages/identity-management-service/locales/en.json` | i18n strings (add keys here for new messages) |
| `apps/identity-management-node/src/identityManagement.ts` | Extension bootstrap |
| `apps/identity-management-node/.env` | Local dev env vars |

## Environment Variables

```bash
IDENTITY_PORT=3001
IDENTITY_TEST_DEPLOYMENT_ENABLED=true   # exposes test routes — NEVER in production
IDENTITY_STORAGE_FILE_ROOT="../../.local-data/identity-entity-storage/"
IDENTITY_PASSWORD_MIN_ENTROPY=120       # bits
```

## Error Handling Pattern

```typescript
import { GeneralError } from '@twin.org/core';
import { ForbiddenError } from './errors/ForbiddenError';

// Business logic errors (400/404/409)
throw new GeneralError('IdentityManagementService', 'agentNotFound', { agentId });

// Permission errors (403)
throw new ForbiddenError('IdentityManagementService', 'insufficientPermissions');
```

## Pre-Submit Checklist

- [ ] `npm run dist:no-test` passes
- [ ] `npm run test` passes
- [ ] `await this.ensureEnforcerReady()` called before any enforcer access
- [ ] No role name strings hardcoded in comparisons
- [ ] Entity mutations use new variables (no direct mutation of stored entity)
- [ ] New locale strings added to `locales/en.json`
