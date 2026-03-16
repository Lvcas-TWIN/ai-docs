---
name: identity-product-design
description: Product design, UI patterns, and design token usage for the identity-mvp project. Use whenever making visual or layout changes, building new UI, or reviewing component usage.
---

# identity-mvp Product Design Guide

## Design System

The project uses `@twin.org/ui-components-react` as the base component library with a custom Tailwind theme that extends Twin.org's tokens.

## Page Layout (Canonical — must match)

All protected pages share this exact outer layout:

```tsx
<div className="max-w-6xl mx-auto px-6 flex flex-col gap-y-8 pt-16">
    <div className="flex flex-col gap-y-2">
        <h1 className="text-3xl font-bold text-brand-primary">{t('title')}</h1>
        <p className="text-base text-tertiary">{t('description')}</p>
    </div>
    {/* section heading + action button */}
    <div className="flex justify-between items-end mb-2">
        <h3 className="text-lg font-semibold text-surface-brand-secondary-1">{t('section')}</h3>
        <Button color={ButtonColors.Secondary}>+ Add</Button>
    </div>
    <PaginatedTable<T> ... />
</div>
```

Reference file: `app/[locale]/(protected)/team/page.tsx`

## Spacing Conventions

| Property | Value |
|---|---|
| Page top padding | `pt-16` |
| Page horizontal | `px-6` |
| Page max width | `max-w-6xl mx-auto` |
| Section gap | `gap-y-8` |
| Heading+subtitle gap | `gap-y-2` |
| Form fields | `space-y-4` or `space-y-6` |
| Card padding | `p-6` |

## Design Tokens

### Text Colours
| Token | Use |
|---|---|
| `text-primary` | Primary/darkest body text |
| `text-secondary` | Secondary body text |
| `text-tertiary` | Muted/helper text, page subtitles |
| `text-brand-primary` | Orange — page headings, active nav, links |
| `text-brand-secondary` | Blue — used in StepView |
| `text-surface-brand-secondary-1` | Blue — section/table headings (h3) |
| `text-success` | Green success state |

### Background Colours
| Token | Use |
|---|---|
| `bg-surface-main` | White surfaces |
| `bg-surface-second` | Page content areas, card backgrounds |
| `bg-surface-third` | Borders, dividers |
| `bg-brand-primary` | Orange brand highlight |
| `bg-system-error-tints-25` | Error/danger sections (light red) |

### Custom Colours (tailwind.config.ts)
| Token | Use |
|---|---|
| `coral-400` | Danger section borders |
| `coral-600` | Danger text / button borders |
| `coral-700` | Danger button text |

### ❌ Tokens That Do NOT Exist
- `surface-bg-first` → use `bg-surface-second`

## Components

### Buttons
```tsx
import { Button, ButtonColors } from '@twin.org/ui-components-react';

<Button color={ButtonColors.Secondary}>Primary action</Button>  // orange
<Button color="plain">Cancel / secondary action</Button>
<Button color="ghost" outline>Outlined ghost</Button>

// Add button — always this exact convention, no leftIcon:
<Button color={ButtonColors.Secondary}>+ Add</Button>
```

### Text Inputs — always sizing="lg"
```tsx
import { TextInput } from '@/components/ui-extension';  // project wrapper

<TextInput
    id="field"
    label={t('label')}
    sizing="lg"
    requiredLabel          // shows "Required" badge
    {...register('field')}
    color={errors.field ? 'failure' : ''}
    helperText={errors.field?.message}
/>
```

### Select
```tsx
import { Select } from '@twin.org/ui-components-react/select';
<Select
    options={[{ value: '', label: t('choose') }, ...]}
    sizing="lg"
/>
```

### Modal
```tsx
import { Modal } from '@twin.org/ui-components-react';
<Modal
    size="lg"
    show={isOpen}
    header="Modal Title"
    onClose={() => setIsOpen(false)}
    body={<div>...</div>}
    footerButtons={[
        { label: t('confirm'), onClick: handleConfirm, variant: 'secondary' },
        { label: t('cancel'), onClick: () => setIsOpen(false), variant: 'ghost', outline: true },
    ]}
/>
```

### Icons
```tsx
import { ArrowRight, Plus, Check, UserCircle, Lock } from '@twin.org/ui-components-react/icons';
// Always specify type and size:
<ArrowRight type="bold" width={20} height={20} />
<Plus type="fill" width={20} height={20} />
```

### PaginatedTable
```tsx
import { PaginatedTable } from '@/components/paginated-table';
const columns: ColumnDef<T>[] = [
    { accessorKey: 'name', header: t('table.name') },
    {
        id: 'actions',
        header: t('table.actions'),
        cell: ({ row }) => (
            <Button color={ButtonColors.Secondary} onClick={() => handleClick(row.original)}>
                {tActions('view')}
            </Button>
        ),
    },
];
<PaginatedTable<T>
    columns={columns}
    queryKey={['myData', id]}
    queryFn={(cursor, pageSize) => fetchFn(cursor, pageSize)}
    placeholderData={serverData}
    getRowId={(row) => row.id}
    emptyMessage={tCommon('emptyState')}
/>
```

### Error Display
```tsx
// Inline form error
<div className="p-4 bg-red-50 border border-red-200 rounded-lg">
    <h3 className="text-sm font-medium text-red-800">{t('errorTitle')}</h3>
    <p className="mt-1 text-sm text-red-700">{error}</p>
</div>

// Danger section (account deletion, destructive actions)
<div className="border border-coral-400 rounded-lg p-4 bg-system-error-tints-25">
    ...
</div>
```

### Loading States
```tsx
import { LoadingSpinner } from '@/components/loading/LoadingSpinner';
<LoadingSpinner size="lg" />

// Skeleton:
<div className="animate-pulse">
    <div className="h-6 bg-gray-200 rounded w-3/4" />
</div>
```

## StepView (Onboarding)

Used only in onboarding steps. All variants come from `@twin.org/ui-components-react`.

### Variants
| Variant | Use |
|---|---|
| `StepViewVariants.Default` | Standard form steps with fields |
| `StepViewVariants.Info` | Informational / confirmation screens with icon |
| `StepViewVariants.Selection` | Choose between options (account type) |
| `StepViewVariants.KYB` | KYC/KYB verification with dynamic field inputs |

### Usage Pattern
```tsx
import { StepView, StepViewVariants } from '@twin.org/ui-components-react';
import { getStepImage, getOnboardingProgress } from '@/lib/utils/onboarding-step-view';
import { useOnboarding } from '@/contexts/OnboardingContext';

const { completeAndNavigate, isSubmitting, state } = useOnboarding();

<StepView
    variant={StepViewVariants.Info}
    title={t('title')}
    description={t('description')}
    image={getStepImage('STEP_ID')}
    progress={getOnboardingProgress(state.currentStep, state.stepStatus)}
    text={<p>...</p>}
    action={{
        label: isSubmitting ? commonT('processingButton') : t('next'),
        onClick: handleNext,
        disabled: isSubmitting,
        loading: isSubmitting,
        loadingText: commonT('processingButton'),
        dataTestId: 'continue-button',
    }}
/>
```

### KYB Variant (dynamic fields)
```tsx
<StepView
    variant={StepViewVariants.KYB}
    title={t('title')}
    progress={...}
    fields={[
        {
            name: 'otp',
            label: t('otpLabel'),
            type: 'verificationCodeInput',
            verificationCodeLength: 4,   // ALWAYS 4 — never omit (default is 6)
            verificationCodeExpiresInText: expiryText,
            dataTestId: 'otp-input',
        },
    ]}
    kybButtons={[
        { label: t('verify'), onClick: handleVerify, variant: 'secondary', loading: isSubmitting },
    ]}
/>
```

## Auth Pages

All auth pages use `AuthLayout`:
```tsx
import { AuthLayout } from '@/components/auth-layout/AuthLayout';
// Split layout: ~30% image / ~70% content
```

## Navigation — Always Locale-Aware

```tsx
// ✅ Always
import { Link, useRouter, usePathname } from '@/i18n/navigation';

// ❌ Never
import Link from 'next/link';
import { useRouter } from 'next/navigation';
```

## Image Upload Preview (blob URLs)

```tsx
// Use plain <img>, NOT next/image — blob URLs not supported
// eslint-disable-next-line @next/next/no-img-element
<img src={URL.createObjectURL(file)} alt="preview" className="..." />
```

## data-testid Conventions

| Element | Pattern |
|---|---|
| Form inputs | `data-testid="email-input"`, `data-testid="password-input"` |
| Submit / continue buttons | `data-testid="continue-button"`, `data-testid="submit-button"` |
| Error messages | `data-testid="login-error"`, `data-testid="verification-error"` |
| Modal triggers | `data-testid="add-member-button"` |
| Table rows | `data-testid="table-row-{id}"` |
| Key sections | `data-testid="profile-content"`, `data-testid="members-table"` |

## Checklist Before Submitting Visual Changes

- [ ] Matches canonical layout (`max-w-6xl mx-auto px-6 pt-16`)
- [ ] All text uses design tokens (no hardcoded colours)
- [ ] `sizing="lg"` on all form inputs
- [ ] No hardcoded UI strings — all in `en.json` + `es.json`
- [ ] `data-testid` on all interactive elements
- [ ] No imports from `next/link` or `next/navigation`
- [ ] Screenshot reviewed and approved before marking complete
