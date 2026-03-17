---
name: identity-business-logic
description: Business logic, ABAC permissions, role hierarchy, credential flows, and core domain rules for the identity-mvp project. Use when implementing or debugging permission checks, role management, credential workflows, or any domain-specific behavior.
---

# identity-mvp Business Logic Guide

## ABAC Permission System

Permissions are computed by `PolicyEnforcer` (Casbin wrapper) per request. They are resource+action boolean maps.

### How It Works
1. Casbin loads `default-policy.csv` (+ any `policyPaths` extensions) at startup
2. `buildRoleRank()` derives numeric ranks from the `g` (grouping) rules
3. Per request: `enforce(userId, orgId, resource, action)` → boolean
4. `getPermissionsForUser(userId, orgId)` → full map of all resource/action pairs

### PolicyEnforcer Public API
```typescript
class PolicyEnforcer {
    initialize(): Promise<void>
    getLowestRole(): string             // e.g. 'member'
    getHighestRole(): string            // e.g. 'primary-owner'
    getRoleRank(): Record<string, number>
    rankOf(role: string): number
    getAssignableRoles(): string[]      // all roles except primary-owner
    getUserRole(userId: string, orgId: string): Promise<string | undefined>
    enforce(userId, orgId, resource, action): Promise<boolean>
    enforceOrThrow(source, userId, orgId, resource, action, errorCode?): Promise<string>
    getPermissionsForUser(userId, orgId): Promise<Record<string, Record<string, boolean>>>
}
```

### Permission Map Shape
```typescript
type PermissionMap = Record<string, Record<string, boolean>>;
// { 'template': { 'list': true, 'create': false }, 'invitation': { 'create': true } }

type RoleRankMap = Record<string, number>;
// { 'primary-owner': 3, 'owner': 2, 'admin': 1, 'member': 0 }
// Higher number = higher privilege
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

### Using Permissions (Server-Side — Next.js page.tsx)
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

const canCreate = permissions?.template?.create ?? false;
const canEdit   = permissions?.['agent-role']?.update ?? false;
```

### Using Permissions (Client-Side — React hook)
```typescript
import { usePermissions } from '@/hooks/usePermissions';

const { permissions, roleRank, hasPermission, isLoading } = usePermissions({
    organizationId,
    initialData,   // SSR data to avoid loading flash
});

hasPermission('template', 'create')    // boolean
hasPermission('agent-role', 'update')  // boolean
```

## Role Hierarchy

### Rules — NEVER violate
- **Never hardcode role names** (`'admin'`, `'member'`) in logic — names come from policy
- Use `rankOf(role)` for numeric comparison
- Higher rank = more permissions
- `isPrivilegedRole = rankOf(role) > 0` (above the lowest)

### Default Role Ladder
```
primary-owner (rank 3) ← only one per org, cannot be removed
owner         (rank 2)
admin         (rank 1)
member        (rank 0) ← lowest, root of hierarchy
```

### Role Change Guard — `isRoleChangeAllowed()`
Two conditions BOTH must be true:
1. Actor rank **strictly greater** than target agent's **current** role rank
2. Actor rank **greater than or equal to** the **new** role rank being assigned

```typescript
const actorRank  = rankOf(actorRole);
const targetRank = rankOf(targetCurrentRole);
const newRank    = rankOf(newRole);

// 1. Can the actor edit this target?
const canEdit    = actorRank > targetRank;  // strictly greater

// 2. Can the actor assign this new role?
const canAssign  = actorRank >= newRank;    // equal allowed (can reassign own rank to others)
```

### Role Utility Functions (frontend — `lib/utils/role.utils.ts`)
```typescript
maxRank(roleRank: RoleRankMap): number
// Highest privilege level in the org

invitableRoles(roleRank: RoleRankMap, currentUserRole: string): string[]
// Roles this user can invite (rank ≤ own AND < max)

assignableRoles(roleRank: RoleRankMap, currentUserRole: string): string[]
// Roles assignable to others (rank < own AND < max)
// Cannot promote to equal or above yourself
```

## Organization & Agent Domain

### Agent Structure (`IAgent`)
```typescript
interface IAgent {
    id: string                    // DID (primary key)
    identifier: string
    email: string                 // Secondary key
    firstName?: string
    lastName?: string
    jobTitle?: string
    preferredLocale?: string
    memberOf: IAgentMemberOf[]    // ONE entry per organization
    affiliation: IGroup[]
    dateCreated: string
    dateModified: string
}

interface IAgentMemberOf {
    organizationId: string
    role: string                  // Single role string — NOT an array
    membershipStatus: 'active' | 'pending' | 'suspended'
}
```

**Important:** An agent has exactly ONE role per organization. `memberOf` is an array of org memberships, not an array of roles.

### Organization Structure (`IOrganization`)
```typescript
interface IOrganization {
    id: string                             // DID (primary key)
    identifier: string
    description?: string
    roleName: IOrganizationRoleName        // 'issuer' | 'holder' | 'issuer-holder'
    classification?: string
    associatedDomains: string[]
    location?: ILocation
    contactPoint?: IContactPoint
    organizationIds?: IOrganizationIdentifier[]  // LEI, EORI, BRN
    dateCreated: string
    dateModified: string
}

type IOrganizationRoleName = 'issuer' | 'holder' | 'issuer-holder';
// issuer     — create templates, issue credentials
// holder     — receive and present credentials
// issuer-holder — both
```

### Organization Identifiers
| Field | Purpose |
|---|---|
| LEI | Legal Entity Identifier (global) |
| EORI | Economic Operators Registration and Identification (EU trade) |
| BRN | Business Registration Number (Kenya KRA) — triggers KYC flow |

## Onboarding Business Rules

### Account Type Choices
```typescript
type AccountType = 'issuer' | 'holder' | 'issuer-holder';
// Set once during onboarding; determines feature availability post-onboarding
```

### Step Order & Dependencies (Backend)
```
1. EMAIL_VERIFICATION          (no deps)
2. ACCOUNT_TYPE                (depends on EMAIL_VERIFICATION)
3. PERSONAL_DETAILS            (depends on ACCOUNT_TYPE)
4. ORGANIZATION_DETAILS        (depends on PERSONAL_DETAILS)
5. ORGANIZATION_DETAILS_EXTRA  (depends on ORGANIZATION_DETAILS)
6. KYC_KRA_KENYA               (optional — only for Kenya BRN)
7. COMPLETED                   (all required steps done)
```

**Note:** `EMAIL_CONFIRMATION` and `INTRODUCTION` are frontend-only UX steps (no backend dependency validation on them).

### Step Statuses
```typescript
ONBOARDING_STEP_STATUS = {
    PENDING:   'pending',
    COMPLETED: 'completed',
    SKIPPED:   'skipped'      // optional steps only
}
```

### Onboarding State Helper Functions
```typescript
areStepDependenciesSatisfied(stepId, stepStatus): boolean
initializeStepStatus(status?, additionalStepKeys?): StepStatusMap
getCompletedStepStatuses(): StepStatusMap
```

## Credential Workflow

### Flow: Template → Request → Approval
1. Privileged user (admin+) **creates a template** (credential schema)
2. An agent **submits a credential request** against a template
3. Privileged user **approves** or **rejects** the request
4. Approved request → **issued credential** (W3C Verifiable Credential as JWT)

### Credential Request Status
```typescript
type CredentialRequestStatus = 'pending' | 'approved' | 'rejected' | 'cancelled';
```

### Template Structure
```typescript
interface ITemplate {
    id: string               // UUID
    organizationId: string
    name: string
    description: string
    credentialType: string
    version: number
    active: boolean          // deactivated templates reject new requests
    schemaUrl: string
    fields: ITemplateField[]
    validityPeriod: IValidityPeriod
    schema: object           // JSON Schema
    nextRevocationIndex: number
    createdBy: string
    createdAt: string
    updatedAt: string
}
```

### Issued Credential
```typescript
interface IIssuedCredential {
    id: string               // Internal UUID
    credentialId: string     // Public W3C VC ID (URL)
    templateId: string
    subjectId: string        // Subject DID
    issuerId: string         // Issuer org DID
    credential: IDidVerifiableCredential   // IMMUTABLE after issuance
    jwt: string                            // IMMUTABLE after issuance
    revocationIndex: number
    revoked: boolean
    expiresAt: string
}
```

### Permission Gates
| Action | Required Permission |
|---|---|
| Create template | `template.create` |
| Edit template | `template.edit` |
| Deactivate template | `template.delete` |
| Approve credential request | `credential-request.approve` |
| Reject credential request | `credential-request.reject` |
| View credential requests | `credential-request.view` |
| Invite agents | `invitation.create` |
| Remove agent | `agent.remove` |
| Change agent role | `agent-role.update` |
| Organization settings | `organization.settings` |
| Delete organization | `organization.delete` |

## Service Methods Reference

### Authentication & Account
```typescript
login(email, password, options?) → { userId, token, tokenExpiry }
verifyToken(email, token) → { isValid, email, onboardingId, onboardingToken, tokenExpiry }
changePassword(currentPassword, newPassword) → { message }
requestPasswordReset(email, options?) → { message }
completePasswordReset(token, newPassword) → { message }
deleteAccount(password) → { message }
```

### Onboarding
```typescript
createOnboarding(email, altcha?, options?) → { onboardingId, message, status, ... }
getOnboardingState(onboardingId) → { onboardingState, completedVerifications }
updateOnboardingState(onboardingId, currentStep, formData, stepStatus, isComplete)
completeOnboarding(onboardingId) → { organizationId, userId, message }
```

### Agent Management
```typescript
getAgent(agentId) → { agent: IAgent }
updateAgent(agentId, data) → { agent: IAgent }
getAgentsByOrganization(organizationId, cursor?, limit?, filters?) → { agents, cursor? }
updateAgentRole(organizationId, agentId, role) → { agent, message }
removeAgentFromOrganization(organizationId, agentId) → { message }
```

### Organization Management
```typescript
getOrganization(organizationId) → { organization: IOrganization }
updateOrganization(organizationId, data) → { organization }
deleteOrganization(organizationId, password) → { message }
getPermissionsForUser(organizationId) → { permissions: PermissionMap }
```

### Templates
```typescript
createTemplate(organizationId, data) → { template }
getTemplate(templateId) → ITemplate
updateTemplate(templateId, data) → { template }
listTemplates(organizationId, cursor?, limit?) → { templates, cursor? }
deactivateTemplate(templateId) → void
getSchema(templateId) → object
```

### Credentials
```typescript
submitCredentialRequest(templateId, subjectData, options?) → { credentialRequestId, message }
approveCredentialRequest(credentialRequestId) → { credentialId, message }
rejectCredentialRequest(credentialRequestId, reason) → void
cancelCredentialRequest(credentialRequestId) → void
getCredentialRequest(credentialRequestId) → ICredentialRequest
listCredentialRequests(organizationId, cursor?, limit?, filters?) → { credentialRequests, cursor? }
getCredential(credentialId) → IIssuedCredential
listCredentials(cursor?, limit?, filters?) → { credentials, cursor? }
revokeCredential(credentialId) → void
verifyCredential(jwt) → { verified, revoked, errors? }
```

### Invitations & Join Requests
```typescript
inviteAgent(organizationId, email, role, options?) → { invitationId, message }
acceptInvitation(token, onboardingData) → { organizationId, userId, message }
requestToJoin(organizationId, options?) → { joinRequestId, message }
approveJoinRequest(joinRequestId, role) → { agentId, message }
```

### KYC Verification
```typescript
verify(onboardingId, providerId, verificationData) → ICompletedVerification
getAvailableVerificationProviders(countryCode) → { providers }
isVerificationRequiredForCountry(countryCode) → boolean
```

### ALTCHA Challenge
```typescript
getChallenge() → { algorithm, challenge, maxnumber, salt, signature }
```

## Error Codes

### GeneralError Codes
```
accessDenied                    – User lacks required permissions
agentNotFound
altchaNotConfigured
cannotChangeOwnRole
cannotDeletePrimaryOwnerAccount – Must first assign primary-owner to another member
cannotGoBackToCompletedStep
cannotRemovePrimaryOwner
credentialAlreadyRevoked
credentialRequestNotPending
duplicateFieldName
envVariablesMisconfiguration
incompleteOnboardingData
insufficientPermissions         – RBAC/ABAC denial
invalidExpirationDate
invalidFieldLabel
invalidFieldName
invalidStep
invitationRankNotAllowed
nodeIdentityNotConfigured
organizationNotFound
passwordResetTokenAlreadyUsed
passwordResetTokenExpired
reservedFieldName
roleAlreadyAssigned
roleChangeNotAllowed
stepDependenciesNotMet
templateAlreadyDeactivated
templateHasPendingRequests
templateNotActive
```

### PolicyEnforcer Error Codes
```
circularRoleHierarchy
noRolesFound
noRootRole
notInitialized
orgSpecificDomainForbidden      – Policy domains must be "*" (not org-specific)
roleHierarchyNotLinear          – Role graph must be a single linear chain
unexpectedRootCount
unknownRole
```

## Authentication & Session (Frontend)

```typescript
// Server-side — get authenticated agent
import { getAuthenticatedAgent } from '@/lib/auth/getAuthenticatedAgent';
const { agent, organization } = await getAuthenticatedAgent();
// Throws/redirects on unauthenticated
```

### Token Details
- JWT format with configurable TTL (`tokenTtlMinutes`, default 60)
- Verified via `verifyToken(email, token)`
- NOT interchangeable with onboarding tokens

## Data Fetching Patterns (Frontend)

| Context | Method |
|---|---|
| Server component / page.tsx | `database.*` directly |
| Client component | TanStack Query hook in `hooks/` |
| API call from client | `apiFetch` from `@/lib/utils/api-fetch` |

## Forms & Validation (Frontend)

All forms use react-hook-form + Zod. Schemas in `lib/schemas/`:

| File | Validates |
|---|---|
| `onboarding.schemas.ts` | Personal details, org details, org details extra |
| `verification.schemas.ts` | BRN, KRA PIN, OTP |
| `registration.schemas.ts` | Email, password |
| `auth.schemas.ts` | Login credentials |
| `template.schemas.ts` | Credential template creation |

**Phone validation:** Use `telephoneRegex` from `lib/constants/` — never write custom phone regex.

## Error Handling (Frontend)

```typescript
import { logError } from '@/lib/utils/error-logging';

try { /* ... */ } catch (error) {
    logError(error, 'ComponentName: descriptive context');
}
```

## Checklist Before Submitting Business Logic Changes

- [ ] No hardcoded role names in comparisons
- [ ] Permission checks use `?? false` fallback
- [ ] Role editing guards use strict `>` for actor-vs-target comparison
- [ ] `ensureEnforcerReady()` awaited before any PolicyEnforcer use
- [ ] Entity mutations use new variables (no direct mutation of stored entities)
- [ ] Server-side data access uses `database.*` only
- [ ] Client-side mutations use `apiFetch` via TanStack Query hook
- [ ] Errors caught and logged with `logError`
- [ ] Zod schema used for any user input validation
