---
name: identity-onboarding
description: Complete guide to the TWIN Identity onboarding system for external integrators. Use when building, modifying, or debugging any onboarding step, integrating KYC flows, or understanding how the onboarding state machine works.
---

# TWIN Identity Onboarding Integration Guide

## Overview

The onboarding system is a multi-step flow that guides a new user from email verification through account creation, organization setup, and optional KYC verification. It uses a state machine stored server-side, synchronized via React Query.

**Route group:** `app/[locale]/(auth)/onboarding/`

## Step Sequence

| Order | Step ID | Route | Required? |
|---|---|---|---|
| 1 | `EMAIL_VERIFICATION` | `/onboarding/email-verification` | Yes |
| 2 | `EMAIL_CONFIRMATION` | `/onboarding/email-confirmation` | Yes |
| 3 | `PERSONAL_DETAILS` | `/onboarding/personal-details` | Yes |
| 4 | `INTRODUCTION` | `/onboarding/introduction` | Yes |
| 5 | `ACCOUNT_TYPE` | `/onboarding/account-type` | Yes |
| 6 | `ORGANIZATION_DETAILS` | `/onboarding/organization-details` | Yes |
| 7 | `ORGANIZATION_DETAILS_EXTRA` | `/onboarding/organization-details-extra` | Yes |
| 8 | `KYC_KRA_KENYA` | `/onboarding/kyc-kra-kenya` | **Optional** |
| 9 | `COMPLETED` | `/onboarding/completed` | Yes |

The `KYC_KRA_KENYA` step is injected by the backend only for Kenya-registered organizations when KYC is enabled. Never assume it is present.

## Constants

```typescript
// lib/constants/onboarding.constants.ts
import { ONBOARDING_ROUTES, STEP_IDS } from '@/lib/constants/onboarding.constants';

STEP_IDS.EMAIL_VERIFICATION     // 'EMAIL_VERIFICATION'
STEP_IDS.PERSONAL_DETAILS       // 'PERSONAL_DETAILS'
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
    stepStatus: IStepStatus;  // Record<OnboardingStepId, StepStatus>
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

### Step Status Values (`lib/types/step.types.ts`)
```typescript
const STEP_STATUS = {
    PENDING: 'pending',
    IN_PROGRESS: 'in_progress',
    COMPLETED: 'completed',
    SKIPPED: 'skipped',
}
```

## OnboardingContext

All onboarding step pages consume a single context. Wrap your step in the `OnboardingProvider`.

```typescript
// contexts/OnboardingContext.tsx
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

### Navigation Method
```typescript
// Complete current step and move to the next
await completeAndNavigate(STEP_IDS.PERSONAL_DETAILS, { personalDetails: formValues });

// Complete final step (sets password and creates account)
await completeOnboarding(passwordValue);

// Skip optional steps
await skipStep(STEP_IDS.KYC_KRA_KENYA);
```

## API Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/api/onboarding` | Fetch current onboarding state |
| `PUT` | `/api/onboarding` | Update onboarding state |
| `POST` | `/api/complete-onboarding` | Final completion with password |
| `GET` | `/api/verification-providers` | List KYC providers for a country |
| `POST` | `/api/verify-id` | Submit KYC verification (BRN or OTP) |

## Building an Onboarding Step

### Standard Step Pattern (Info variant)
```tsx
// app/[locale]/(auth)/onboarding/my-step/page.tsx
import { StepView, StepViewVariants } from '@twin.org/ui-components-react';
import { getStepImage, getOnboardingProgress } from '@/lib/utils/onboarding-step-view';
import { useOnboarding } from '@/contexts/OnboardingContext';
import { useTranslations } from 'next-intl';
import { STEP_IDS } from '@/lib/constants/onboarding.constants';

export default function MyStep() {
    const { completeAndNavigate, isSubmitting, state } = useOnboarding();
    const t = useTranslations('onboarding.myStep');
    const commonT = useTranslations('common');

    const handleNext = async () => {
        await completeAndNavigate(STEP_IDS.MY_STEP);
    };

    return (
        <StepView
            variant={StepViewVariants.Info}
            title={t('title')}
            description={t('description')}
            image={getStepImage('MY_STEP')}
            progress={getOnboardingProgress(state.currentStep, state.stepStatus)}
            action={{
                label: isSubmitting ? commonT('processingButton') : t('next'),
                onClick: handleNext,
                disabled: isSubmitting,
                loading: isSubmitting,
                loadingText: commonT('processingButton'),
                dataTestId: 'continue-button',
            }}
        />
    );
}
```

### Form Step Pattern (Default variant)
```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { mySchema, type MySchema } from '@/lib/schemas/my.schemas';

export default function MyFormStep() {
    const { completeAndNavigate, isSubmitting, state } = useOnboarding();
    const { register, handleSubmit, formState: { errors } } = useForm<MySchema>({
        resolver: zodResolver(mySchema),
        defaultValues: state.formData.myField ?? undefined,
    });
    const t = useTranslations('onboarding.myFormStep');

    const onSubmit = async (data: MySchema) => {
        await completeAndNavigate(STEP_IDS.MY_FORM_STEP, { myField: data });
    };

    return (
        <StepView
            variant={StepViewVariants.Default}
            title={t('title')}
            image={getStepImage('MY_FORM_STEP')}
            progress={getOnboardingProgress(state.currentStep, state.stepStatus)}
            text={
                <form onSubmit={handleSubmit(onSubmit)}>
                    <TextInput
                        id="fieldName"
                        label={t('fieldLabel')}
                        sizing="lg"
                        requiredLabel
                        {...register('fieldName')}
                        color={errors.fieldName ? 'failure' : ''}
                        helperText={errors.fieldName?.message}
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

## Progress Bar

Always pass both arguments — it handles optional steps and sub-steps automatically:

```typescript
import { getOnboardingProgress } from '@/lib/utils/onboarding-step-view';

const progress = getOnboardingProgress(
    state.currentStep,   // current step ID
    state.stepStatus,    // full step status map
);
// Returns 0–100 (number)
```

## Step Image

```typescript
import { getStepImage } from '@/lib/utils/onboarding-step-view';

getStepImage('EMAIL_VERIFICATION')
// Returns: '/onboarding/email-verification.webp'
```
Images are stored in `/public/onboarding/` as `.webp` files. Add a new image there when adding a new step.

## KYC KRA Kenya Step (Complex Multi-Stage)

The KYC step is a self-contained multi-stage flow within a single page. Only present for Kenya orgs.

### Stages
1. **Prompt** — displays requirements (`StepViewVariants.Info`)
2. **Business Verification** — collects BRN + KRA PIN (`StepViewVariants.KYB`)
3. **OTP** — 4-digit OTP verification (`StepViewVariants.KYB`)
4. **Result** — success or failure (`StepViewVariants.Info`)

### Dynamic Sub-Steps
The backend may inject a `KYC_KRA_OTP` dynamic sub-step into `stepStatus` when OTP is required. The progress bar accounts for this automatically via `getOnboardingProgress`.

### OTP Input — Always 4 digits
```typescript
// verificationCodeLength MUST be explicitly set — default is 6
{
    name: 'otp',
    label: t('otpLabel'),
    type: 'verificationCodeInput',
    verificationCodeLength: 4,   // hardcoded, never dynamic
    verificationCodeExpiresInText: expiryText,
    dataTestId: 'otp-input',
}
```

### Verification Result Types
```typescript
interface IVerificationResult {
    success: boolean;
    error?: string;
    message?: string;
    data?: Record<string, unknown>;
    entityName?: string;
    verifiedId?: string;
    phaseComplete?: boolean;  // true when all providers verified
}

interface ICompletedVerification {
    providerId: string;
    verifiedId: string;
    entityName?: string;
    verifiedAt: string;  // ISO timestamp
}
```

### Verification Hooks
```typescript
// Fetch available KYC providers for the organization's country
const { data: providers } = useVerificationProviders(organizationCountry);

// Submit a verification (BRN or OTP)
const { mutate: verifyId } = useVerifyId();
verifyId({ providerId, type: 'BRN' | 'OTP', value: inputValue });
```

## Adding a New Onboarding Step

1. **Add step ID** to `OnboardingStepId` union in `lib/interfaces/IOnboardingState.ts`
2. **Add route constant** to `ONBOARDING_ROUTES` and `STEP_IDS` in `lib/constants/onboarding.constants.ts`
3. **Add to step sequence** in `lib/interfaces/IOnboardingState.ts` (step order array)
4. **Add step status entry** in `createInitialStepStatus()` in `lib/utils/onboarding.utils.ts`
5. **Add data key mapping** in `stepDataKeyMap` if the step stores form data
6. **Create page** at `app/[locale]/(auth)/onboarding/[step-name]/page.tsx`
7. **Add step image** at `public/onboarding/[kebab-step-name].webp`
8. **Add i18n keys** to `en.json` and `es.json` under `onboarding.[stepName]`
9. **Update backend** to include the new step in its state machine

## i18n Key Structure

```json
// messages/en.json
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
**Always update both `en.json` AND `es.json` together.**

## Utility Functions (`lib/utils/onboarding.utils.ts`)

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

## Checklist for New Onboarding Steps

- [ ] Step ID added to `OnboardingStepId` union
- [ ] Route and step constant added
- [ ] Step included in step order array
- [ ] Initial status entry added
- [ ] Page created at correct route
- [ ] Uses `useOnboarding()` context
- [ ] Uses `getOnboardingProgress()` for progress bar
- [ ] Uses `getStepImage()` for image
- [ ] Uses `completeAndNavigate()` (never manual routing)
- [ ] `dataTestId: 'continue-button'` on main action
- [ ] Both `en.json` and `es.json` updated
- [ ] Image `.webp` file added to `/public/onboarding/`
