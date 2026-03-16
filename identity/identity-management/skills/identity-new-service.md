---
name: identity-new-service
description: End-to-end guide for integrating a new external team or service with the TWIN Identity system. Use when onboarding a new team (like SSCA), adding custom roles, exposing credential services to external consumers, or writing integration specs.
---

# TWIN Identity — New Service Integration Guide

## Overview

This guide covers everything an external team needs to integrate with the TWIN Identity system:
1. Adding custom roles with specific permissions (via `policyPaths`)
2. Inviting team members with those roles
3. Consuming credential services (templates, requests, approvals)
4. Writing integration specs and tests

**Reference use case:** SSCA team integration (Supply Chain Credential Authority)

## Architecture: How External Teams Plug In

The identity service uses Casbin RBAC with CSV policy files. External teams extend the default roles by providing **additional policy CSV files** at service initialization — without touching the core `default-policy.csv`.

```
default-policy.csv (core roles: member → admin → owner → primary-owner)
     +
ssca-policy.csv (adds: ssca-reviewer, ssca-admin — connected to core chain)
     =
merged policy in Casbin
```

## Step 1 — Write the Policy Spec

Before writing code, specify what your team needs:

### SSCA Integration Spec Example
```
Team: SSCA (Supply Chain Credential Authority)
Purpose: Review and approve credential requests for supply-chain credentials

Roles:
  ssca-reviewer
    - Can view credential requests
    - Can list credentials
    - Can list templates
    - Cannot approve/reject requests
    - Cannot create templates

  ssca-admin
    - Everything ssca-reviewer can do
    - Can approve and reject credential requests
    - Can create and edit templates
    - Can invite other ssca-* members (up to own rank)

Role hierarchy:
  ssca-reviewer inherits from member (rank 0)
  ssca-admin    inherits from admin  (rank 1)

  Full chain after merge:
  member(0) → admin(1) → owner(2) → primary-owner(3)
      ↑            ↑
  ssca-reviewer  ssca-admin
```

## Step 2 — Create the Policy CSV File

```csv
# packages/identity-management-service/src/policies/ssca-policy.csv
# SSCA Integration Policy
# See: docs/integrations/ssca.md

# Role hierarchy — connect to existing core roles
# g, role, parent-role
g, ssca-reviewer, member
g, ssca-admin, admin

# ssca-reviewer permissions
# p, role, domain(*=any org), resource, action
p, ssca-reviewer, *, template, list
p, ssca-reviewer, *, credential-request, view
p, ssca-reviewer, *, credentials, list

# ssca-admin permissions (also inherits admin permissions via hierarchy)
p, ssca-admin, *, template, list
p, ssca-admin, *, template, create
p, ssca-admin, *, template, edit
p, ssca-admin, *, credential-request, view
p, ssca-admin, *, credential-request, approve
p, ssca-admin, *, credential-request, reject
p, ssca-admin, *, invitation, create
```

**Rules for policy CSV:**
- Domain MUST be `*` (org-specific domains are rejected by `orgSpecificDomainForbidden`)
- Each new role MUST connect to the existing chain via `g` rules
- Role hierarchy must remain a single linear chain (no branching)
- Use lowercase kebab-case for role names

## Step 3 — Wire Policy Into Service

```typescript
// apps/identity-management-node/src/identityManagement.ts
import path from 'path';

export async function identityManagementTypeInitialiser(
    engineCore: IEngineCore,
    context: IEngineCoreContext,
    instanceConfig: {...}
) {
    const service = new IdentityManagementService({
        // ...existing config...
        policyPaths: [
            path.resolve('./policies/ssca-policy.csv'),
        ],
    });

    return { component: service };
}
```

Copy the policy file to the dist output — add to build step or place in a directory already copied by the build.

## Step 4 — Invite Team Members

Once the policy is loaded, `ssca-reviewer` and `ssca-admin` are valid roles for invitations:

```typescript
// Via REST API — requires invitation.create permission
POST /identity-management/organizations/:organizationId/invite
{
    "email": "reviewer@ssca.org",
    "role": "ssca-reviewer"
}

POST /identity-management/organizations/:organizationId/invite
{
    "email": "lead@ssca.org",
    "role": "ssca-admin"
}
```

Or programmatically:
```typescript
await service.inviteAgent(orgId, 'reviewer@ssca.org', 'ssca-reviewer');
await service.inviteAgent(orgId, 'lead@ssca.org', 'ssca-admin');
```

Invitation rank rules still apply: the actor cannot invite someone to a rank equal to or above their own.

## Step 5 — Consuming Services (API Calls)

### Authentication
```bash
# 1. Get a token (use the email+password from account creation)
POST /identity-management/login
{ "email": "lead@ssca.org", "password": "..." }
→ { "token": "eyJ...", "tokenExpiry": 1234567890 }

# 2. Include token in all subsequent requests
Authorization: Bearer eyJ...
```

### Listing Templates
```bash
GET /identity-management/templates?organizationId=<orgId>
Authorization: Bearer <token>
→ { "templates": [...], "cursor": "..." }
```

### Listing Credential Requests
```bash
GET /identity-management/credential-requests?organizationId=<orgId>
Authorization: Bearer <token>

# Filter by status
GET /identity-management/credential-requests?organizationId=<orgId>&status=pending
```

### Approving a Credential Request (ssca-admin only)
```bash
POST /identity-management/credential-requests/:credentialRequestId/approve
Authorization: Bearer <token>
→ { "credentialId": "...", "message": "..." }
```

### Rejecting a Credential Request (ssca-admin only)
```bash
POST /identity-management/credential-requests/:credentialRequestId/reject
Authorization: Bearer <token>
{ "reason": "Invalid documentation" }
```

### Checking Your Permissions
```bash
GET /identity-management/organizations/:organizationId/permissions
Authorization: Bearer <token>
→ {
    "permissions": {
        "template": { "list": true, "create": true, "edit": true },
        "credential-request": { "view": true, "approve": true, "reject": true },
        "invitation": { "create": true }
    }
}
```

## Step 6 — Write Integration Tests

```typescript
// tests/integrations/ssca.spec.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { IdentityManagementService } from '../../src/identityManagementService';
import { MemoryEntityStorageConnector } from '@twin.org/entity-storage-connector-memory';
import path from 'path';

describe('SSCA Integration', () => {
    let service: IdentityManagementService;

    beforeEach(async () => {
        service = new IdentityManagementService({
            argon2TimeCost: 1,
            argon2MemoryCost: 1024,
            argon2Parallelism: 1,
            passwordMinEntropy: 50,
            altchaHmacKey: 'test-key',
            policyPaths: [path.resolve('./src/policies/ssca-policy.csv')],
            // ...connectors
        });
        await service.start();
    });

    describe('ssca-reviewer', () => {
        it('can list credential requests', async () => {
            // arrange: create org, add ssca-reviewer member, create a credential request
            const { requests } = await service.listCredentialRequests(orgId);
            expect(requests).toBeDefined();
        });

        it('cannot approve credential requests', async () => {
            // arrange: switch context to ssca-reviewer
            await expect(service.approveCredentialRequest(requestId))
                .rejects.toMatchObject({ properties: { code: 'insufficientPermissions' } });
        });

        it('cannot create templates', async () => {
            await expect(service.createTemplate(orgId, templateData))
                .rejects.toMatchObject({ properties: { code: 'insufficientPermissions' } });
        });
    });

    describe('ssca-admin', () => {
        it('can approve credential requests', async () => {
            const result = await service.approveCredentialRequest(requestId);
            expect(result.credentialId).toBeDefined();
        });

        it('can create templates', async () => {
            const result = await service.createTemplate(orgId, templateData);
            expect(result.template).toBeDefined();
        });

        it('inherits all admin permissions', async () => {
            const { permissions } = await service.getPermissionsForUser(orgId);
            expect(permissions['template']['create']).toBe(true);
            expect(permissions['agent-role']['update']).toBe(true);
        });

        it('cannot modify owner (rank too high)', async () => {
            await expect(service.updateAgentRole(orgId, ownerId, 'ssca-admin'))
                .rejects.toMatchObject({ properties: { code: 'roleChangeNotAllowed' } });
        });
    });

    describe('role hierarchy', () => {
        it('ssca-reviewer rank is between member and admin', async () => {
            const ranks = service.getRoleRank?.() ?? {};
            expect(ranks['ssca-reviewer']).toBeGreaterThan(ranks['member'] ?? 0);
            expect(ranks['ssca-reviewer']).toBeLessThanOrEqual(ranks['admin'] ?? 0);
        });
    });
});
```

## Step 7 — Documentation

Write an integration doc at `docs/integrations/<team-name>.md`:

```markdown
# SSCA Integration

## Purpose
SSCA manages supply-chain credential issuance for logistics providers.

## Roles
| Role | Inherits From | Can Do |
|---|---|---|
| ssca-reviewer | member | view credential requests, list templates |
| ssca-admin | admin | + approve/reject requests, create templates |

## Setup
1. Add `ssca-policy.csv` to service config `policyPaths`
2. Restart the service
3. Invite SSCA team members via admin UI or API

## API Usage
See `/docs/api/credential-requests.md` for endpoint details.
```

## Common Integration Patterns

### Pattern: Read-Only Auditor
```csv
g, auditor, member
p, auditor, *, credential-request, view
p, auditor, *, credentials, list
p, auditor, *, template, list
```

### Pattern: Template Manager (no approval rights)
```csv
g, template-manager, admin
p, template-manager, *, template, list
p, template-manager, *, template, create
p, template-manager, *, template, edit
# Explicitly does NOT get credential-request.approve
```

### Pattern: Approval-Only Reviewer
```csv
g, approver, member
p, approver, *, credential-request, view
p, approver, *, credential-request, approve
p, approver, *, credential-request, reject
```

## Checklist for New Service Integration

- [ ] Integration spec written (roles, permissions, hierarchy diagram)
- [ ] Policy CSV created and tested
- [ ] Policy CSV wired in `identityManagement.ts` via `policyPaths`
- [ ] Role hierarchy is linear (no branching, no cycles)
- [ ] All role names lowercase kebab-case
- [ ] All domains in policy are `*` (not org-specific)
- [ ] Integration tests written covering: allowed actions, denied actions, rank edge cases
- [ ] `npm run test` passes
- [ ] `npm run dist:no-test` passes
- [ ] Integration doc written at `docs/integrations/<name>.md`
- [ ] Team members invited with correct roles
