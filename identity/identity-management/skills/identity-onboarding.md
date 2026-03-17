---
name: identity-onboarding
description: Complete guide to the TWIN Identity onboarding system for external integrators. Use when building, modifying, or debugging any onboarding step, integrating KYC flows, adding new roles via policyPaths, or understanding how the onboarding state machine works.
---

# TWIN Identity Onboarding Integration Guide

## Overview

The onboarding system is a multi-step flow that guides a new user from email verification through account creation, organization setup, and optional KYC verification. It uses a state machine stored server-side, synchronized via React Query on the frontend.

**Route group:** `app/[locale]/(auth)/onboarding/`

## Step Sequence

| Order | Step ID | Route | Required? |
|---|---|---|---|
| 1 | `EMAIL_VERIFICATION` | `/onboarding/email-verification` | Yes |
| 2 | `EMAIL_CONFIRMATION` | `/onboarding/email-confirmation` | Yes (UX only) |
| 3 | `PERSONAL_DETAILS` | `/onboarding/personal-details` | Yes |
| 4 | `INTRODUCTION` | `/onboarding/introduction` | Yes (UX only) |
| 5 | `ACCOUNT_TYPE` | `/onboarding/account-type` | Yes |
| 6 | `ORGANIZATION_DETAILS` | `/onboarding/organization-details` | Yes |
| 7 | `ORGANIZATION_DETAILS_EXTRA` | `/onboarding/organization-details-extra` | Yes |
| 8 | `KYC_KRA_KENYA` | `/onboarding/kyc-kra-kenya` | **Optional** |
| 9 | `COMPLETED` | `/onboarding/completed` | Yes |

The `KYC_KRA_KENYA` step is only injected for Kenya-registered organizations when KYC is enabled in the service config. Never assume it is present.

**Backend dependency order (enforced server-side):**
```
EMAIL_VERIFICATION → ACCOUNT_TYPE → PERSONAL_DETAILS → ORGANIZATION_DETAILS → ORGANIZATION_DETAILS_EXTRA
```

`EMAIL_CONFIRMATION` and `INTRODUCTION` are UX-only steps without backend dependency checks.

## Constants

```typescript
// lib/constants/onboarding.constants.ts
import { ONBOARDING_ROUTES, STEP_IDS } from '@/lib/constants/onboarding.constants';

STEP_IDS.EMAIL_VERIFICATION         // 'EMAIL_VERIFICATION'
STEP_IDS.PERSONAL_DETAILS           // 'PERSONAL_DETAILS'
STEP_IDS.ORGANIZATION_DETAILS_EXTRA // 'ORGANIZATION_DETAILS_EXTRA'
// etc.

ONBOARDING_ROUTES.PERSONAL_DETAILS  // '/onboarding/personal-details'
```

## Onboarding State

```typescript
// lib/interfaces/IOnboardingState.ts
interface IOnboardingState {
    id: string;
    email: string;
    currentStep: OnboardingStepId;
    preferredLocale?: string;
    formData: IOnboardingFormData;
    stepStatus: IStepStatus;   // Record<OnboardingStepId, StepStatus>
    isComplete: boolean;
    dateCreated: string;
    dateModified: string;
}

interface IOnboardingFormData {
    accountType?: 'issuer' | 'holder' | 'issuer-holder' | null;
    personalDetails?: IPersonalDetails | null;
    emailVerified: boolean;
    organizationDetails?: IOrganizationDetails | null;
    organizationDetailsExtra?: IOrganizationDetailsExtra | null;
}
```

### Step Status Values
```typescript
// 'pending' | 'completed' | 'skipped'
ONBOARDING_STEP_STATUS.PENDING    // 'pending'
ONBOARDING_STEP_STATUS.COMPLETED  // 'completed'
ONBOARDING_STEP_STATUS.SKIPPED    // 'skipped'
```

## OnboardingContext

All onboarding step pages consume a single context. Wrap steps in `OnboardingProvider`.

```typescript
import { useOnboarding } from '@/contexts/OnboardingContext';

const {
    state,                  // IOnboardingState — current full state
    updateFormData,         // (data: Partial<formData>) => Promise<void>
    skipStep,               // (step | step[]) => Promise<void>
    completeAndNavigate,    // (step, formData?) => Promise<void>
    completeOnboarding,     // (password: string) => Promise<void>
    adminEmail,             // string
    isLoading,              // boolean
    isError,                // boolean
    isSubmitting,           // boolean
    lastCompletionError,    // { code: string; message: string } | null
    clearCompletionError,   // () => void
} = useOnboarding();
```

### Navigation Methods
```typescript
// Complete current step and move to next
await completeAndNavigate(STEP_IDS.PERSONAL_DETAILS, { personalDetails: formValues });

// Complete final step (sets password, creates account + org)
await completeOnboarding(passwordValue);

// Skip optional step
await skipStep(STEP_IDS.KYC_KRA_KENYA);
```

## API Endpoints

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/onboarding` | Token | Fetch current onboarding state |
| `PUT` | `/api/onboarding` | Token | Update onboarding state |
| `POST` | `/api/complete-onboarding` | Token | Final completion with password |
| `GET` | `/api/verification-providers` | Token | List KYC providers for a country |
| `POST` | `/api/verify-id` | Token | Submit KYC verification (BRN or OTP) |

## Building an Onboarding Step

### Info Step (no form)
```tsx
import { StepView, StepViewVariants } from '@twin.org/ui-components-react';
import { getStepImage, getOnboardingProgress } from '@/lib/utils/onboarding-step-view';
import { useOnboarding } from '@/contexts/OnboardingContext';
import { useTranslations } from 'next-intl';
import { STEP_IDS } from '@/lib/constants/onboarding.constants';

export default function IntroductionStep() {
    const { completeAndNavigate, isSubmitting, state } = useOnboarding();
    const t = useTranslations('onboarding.introduction');
    const commonT = useTranslations('common');

    return (
        <StepView
            variant={StepViewVariants.Info}
            title={t('title')}
            description={t('description')}
            image={getStepImage('INTRODUCTION')}
            progress={getOnboardingProgress(state.currentStep, state.stepStatus)}
            action={{
                label: isSubmitting ? commonT('processingButton') : t('next'),
                onClick: () => completeAndNavigate(STEP_IDS.INTRODUCTION),
                disabled: isSubmitting,
                loading: isSubmitting,
                dataTestId: 'continue-button',
            }}
        />
    );
}
```

### Form Step
```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { mySchema, type MySchema } from '@/lib/schemas/my.schemas';

export default function PersonalDetailsStep() {
    const { completeAndNavigate, isSubmitting, state } = useOnboarding();
    const { register, handleSubmit, formState: { errors } } = useForm<MySchema>({
        resolver: zodResolver(mySchema),
        defaultValues: state.formData.personalDetails ?? undefined,
    });
    const t = useTranslations('onboarding.personalDetails');
    const commonT = useTranslations('common');

    const onSubmit = async (data: MySchema) => {
        await completeAndNavigate(STEP_IDS.PERSONAL_DETAILS, { personalDetails: data });
    };

    return (
        <StepView
            variant={StepViewVariants.Default}
            title={t('title')}
            image={getStepImage('PERSONAL_DETAILS')}
            progress={getOnboardingProgress(state.currentStep, state.stepStatus)}
            text={
                <form onSubmit={handleSubmit(onSubmit)}>
                    <TextInput
                        id="firstName"
                        label={t('firstName')}
                        sizing="lg"
                        requiredLabel
                        {...register('firstName')}
                        color={errors.firstName ? 'failure' : ''}
                        helperText={errors.firstName?.message}
                    />
                </form>
            }
            action={{
                label: isSubmitting ? commonT('processingButton') : t('next'),
                onClick: handleSubmit(onSubmit),
                disabled: isSubmitting,
                loading: isSubmitting,
                dataTestId: 'continue-button',
            }}
        />
    );
}
```

## Progress Bar & Step Image

```typescript
import { getOnboardingProgress } from '@/lib/utils/onboarding-step-view';
import { getStepImage } from '@/lib/utils/onboarding-step-view';

// Always pass both args — handles optional steps and sub-steps automatically
const progress = getOnboardingProgress(state.currentStep, state.stepStatus);  // 0–100

// Images are stored in /public/onboarding/ as .webp files
getStepImage('EMAIL_VERIFICATION')  // returns '/onboarding/email-verification.webp'
```

## KYC KRA Kenya Step (Complex Multi-Stage)

Self-contained multi-stage flow within one page. Only present for Kenya org addresses.

### Stages
1. **Prompt** — displays requirements (`StepViewVariants.Info`)
2. **Business Verification** — collects BRN + KRA PIN (`StepViewVariants.KYB`)
3. **OTP** — 4-digit OTP verification (`StepViewVariants.KYB`)
4. **Result** — success or failure (`StepViewVariants.Info`)

### OTP Input — Always 4 digits, never 6
```typescript
{
    name: 'otp',
    type: 'verificationCodeInput',
    verificationCodeLength: 4,   // HARDCODED — default in the component is 6, must override
    verificationCodeExpiresInText: expiryText,
    dataTestId: 'otp-input',
}
```

### Verification Hooks
```typescript
const { data: providers } = useVerificationProviders(organizationCountry);

const { mutate: verifyId } = useVerifyId();
verifyId({ providerId, type: 'BRN' | 'OTP', value: inputValue });
```

### Dynamic Sub-Step
Backend may inject `KYC_KRA_OTP` into `stepStatus` when OTP is required. The progress bar handles this automatically via `getOnboardingProgress`.

## Adding a New Onboarding Step (Frontend)

1. Add step ID to `OnboardingStepId` union in `lib/interfaces/IOnboardingState.ts`
2. Add route constant to `ONBOARDING_ROUTES` and `STEP_IDS` in `lib/constants/onboarding.constants.ts`
3. Add to step sequence in `lib/interfaces/IOnboardingState.ts`
4. Add status entry in `createInitialStepStatus()` in `lib/utils/onboarding.utils.ts`
5. Add data key mapping in `stepDataKeyMap` if the step stores form data
6. Create page at `app/[locale]/(auth)/onboarding/[step-name]/page.tsx`
7. Add step image at `public/onboarding/[kebab-step-name].webp`
8. Add i18n keys to `en.json` and `es.json` under `onboarding.[stepName]`
9. **Coordinate with backend** to include the new step in service constants

## Extending the System for External Integrators (SSCA Pattern)

External teams can add **new roles** with custom permission sets without modifying core code. The service accepts extra Casbin CSV files via `policyPaths`.

### Step 1 — Create a new policy file
```csv
# ssca-policy.csv

# Role hierarchy: connect your roles to existing ones
g, ssca-reviewer, member    // inherits all member permissions
g, ssca-admin, admin        // inherits all admin permissions

# Additional permissions for ssca-reviewer
p, ssca-reviewer, *, credential-request, view
p, ssca-reviewer, *, credentials, list

# Additional permissions for ssca-admin
p, ssca-admin, *, template, list
p, ssca-admin, *, template, create
p, ssca-admin, *, credential-request, approve
p, ssca-admin, *, credential-request, reject
```

### Step 2 — Wire it in at service initialization
```typescript
// apps/identity-management-node/src/identityManagement.ts
const service = new IdentityManagementService({
    policyPaths: [path.resolve('./policies/ssca-policy.csv')],
    // ...rest of config
});
```

### Step 3 — Invite agents with the new roles
```typescript
// After wiring, ssca-reviewer and ssca-admin are valid roles
await service.inviteAgent(orgId, 'reviewer@ssca.org', 'ssca-reviewer');
await service.approveJoinRequest(joinRequestId, 'ssca-admin');
```

### Constraint: Linear hierarchy required
```
member → admin → owner → primary-owner
  ↑         ↑
ssca-reviewer  ssca-admin
```
Both `ssca-reviewer` and `ssca-admin` connect to the chain (not to each other). The full graph must remain a single linear chain. Branching or cycles throw `roleHierarchyNotLinear`.

### SSCA Runtime Example
When `ssca-reviewer` calls `getPermissionsForUser`:
```json
{
  "credential-request": { "view": true, "approve": false, "reject": false },
  "template": { "list": true, "create": false },
  "invitation": { "create": false },
  "agent-role": { "update": false }
}
```

When `ssca-admin` calls `getPermissionsForUser`:
```json
{
  "credential-request": { "view": true, "approve": true, "reject": true },
  "template": { "list": true, "create": true, "edit": true },
  "invitation": { "create": true },
  "agent-role": { "update": true }
}
```

## Utility Functions (`lib/utils/onboarding.utils.ts` — Frontend)

```typescript
getNextPendingStep(stepStatus): OnboardingStepId | undefined
// Returns next step that is 'pending' or 'in_progress'

hasPendingDynamicKycStep(stepStatus): boolean
// True if a dynamic KYC sub-step is pending

createInitialOnboardingState(email?: string): OnboardingData
// Creates fresh default state for a new user

validateStepOrder(steps): boolean
// Validates step sequence is correct
```

## Backend Helper Functions (`src/constants/onboardingConstants.ts`)

```typescript
areStepDependenciesSatisfied(stepId, stepStatus): boolean
// Returns false if any dependency step is not yet 'completed'

initializeStepStatus(status?, additionalStepKeys?): StepStatusMap
// Creates initial status map (all 'pending' by default)

getCompletedStepStatuses(): StepStatusMap
// All steps marked 'completed' — used for testing/seeding
```

## i18n Key Structure

```json
{
  "onboarding": {
    "myStep": {
      "title": "...",
      "description": "...",
      "next": "Continue"
    }
  }
}
```
Always update both `en.json` AND `es.json` together.

## Checklist for New Onboarding Steps

- [ ] Step ID added to `OnboardingStepId` union
- [ ] Route and step constant added
- [ ] Step included in step order array
- [ ] Initial status entry added in `createInitialStepStatus()`
- [ ] Page created at correct route
- [ ] Uses `useOnboarding()` context — never manual navigation
- [ ] Uses `getOnboardingProgress()` for progress bar
- [ ] Uses `getStepImage()` for image
- [ ] Uses `completeAndNavigate()` — never `router.push()` directly
- [ ] `dataTestId: 'continue-button'` on the main action
- [ ] Both `en.json` and `es.json` updated
- [ ] Image `.webp` added to `/public/onboarding/`
- [ ] Backend service updated with new step constants + dependencies
