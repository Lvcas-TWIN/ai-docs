# CLAUDE.md — identity-mvp Project Rules

> Project-specific rules for Claude. These extend the global `~/.claude/CLAUDE.md`.
> **Keep this file updated whenever structural changes are made to the project.**

---

## Project Overview

TWIN Identity MVP — a Next.js 16 app for managing decentralised identity credentials, onboarding organisations onto the TWIN network, and issuing/verifying verifiable credentials.

**Stack:** Next.js 16 + React 19 + TypeScript 5 + Tailwind CSS 3 + TanStack Query 5 + next-intl 4 + Zod + Vitest + Playwright

---

## Commands

```bash
npm run dev              # Start dev server
npm run build            # Production build (run before every push)
npm run typecheck        # tsc --noEmit (run after every significant change)
npm run lint             # ESLint + Prettier check
npm run lint-fix         # Auto-fix ESLint issues
npm run format           # Prettier write
npm run test:unit        # Vitest unit tests
npm run test:e2e         # Playwright e2e tests
npm run i18n-check       # Validate translation keys vs en.json
```

**Always run `npm run typecheck` after any code change. Run `npm run build` before any push.**

---

## Project Structure

```
app/
  [locale]/
    (auth)/              # Unprotected pages: login, register, reset-password, onboarding/*
    (protected)/         # Authenticated pages: profile, team, credentials/*, settings
  api/                   # API route handlers
  messages/              # i18n: en.json + es.json (always keep in sync)
components/              # Shared components
contexts/                # React Context providers
hooks/                   # Custom React hooks
lib/
  constants/             # App constants (routes, onboarding steps, etc.)
  interfaces/            # TypeScript interfaces
  schemas/               # Zod validation schemas
  database/              # Database singleton
  server/                # Server-only utilities (profile.ts, session.ts)
  utils/                 # Helper functions
  services/              # API service functions
providers/               # React providers (QueryClient, etc.)
e2e/                     # Playwright tests
tests/                   # Vitest unit tests
```

---

## Page Patterns

### Protected Page (canonical pattern — use Team page as reference)

```
app/[locale]/(protected)/[section]/page.tsx        ← Server component: fetch data + permissions
app/[locale]/(protected)/[section]/[Section]PageClient.tsx  ← Client component: UI + interactions
```

**Server `page.tsx`:**
```tsx
export default async function MyPage() {
    const result = await getAuthenticatedAgent();
    if (!result || !result.organization) return null;

    const { agent, organization, role: currentUserRole } = result;

    let permissions: PermissionMap = {};
    try {
        const permResult = await database.getPermissionsForUser(organization.id);
        permissions = permResult.permissions;
    } catch (error) {
        logError(error, 'MyPage: Error fetching permissions');
    }

    const canDoThing = permissions?.resource?.action ?? false;

    return (
        <div className="min-h-screen">
            <div className="max-w-6xl mx-auto px-6 flex flex-col">
                <div className="pt-16 pb-4 flex flex-col items-start">
                    <h1 className="text-3xl font-bold text-brand-primary mb-2">
                        {t('pages.myPage.title')}
                    </h1>
                    <p className="text-base text-tertiary">{t('myPage.subtitle')}</p>
                </div>
                <MyPageClient canDoThing={canDoThing} />
            </div>
        </div>
    );
}
```

**Client `PageClient.tsx`:**
- Wrapper: `max-w-6xl mx-auto px-6 flex flex-col gap-y-8 pt-16`
- Title: `h1 text-3xl font-bold text-brand-primary mb-2`
- Subtitle: `p text-base text-tertiary`
- Section heading: `h3 text-lg font-semibold text-surface-brand-secondary-1`
- Add button: `ButtonColors.Secondary` with `+ Add` label pattern

### Auth Page

All auth pages use `AuthLayout` from `@/components/auth-layout/AuthLayout`:
```tsx
<AuthLayout
    mainTitle={t('mainTitle')}
    title={t('title')}           // optional — makes title large
    subtitle={t('subtitle')}
    imageSrc="/my-image.webp"
    imageAlt={t('imageAlt')}
    backgroundImageSrc="/single-logo.svg"  // optional
>
    <MyFormClient />
</AuthLayout>
```

### Onboarding Step

All onboarding steps use `StepView` from `@twin.org/ui-components-react`:
```tsx
<StepView
    variant={StepViewVariants.Default | Info | Selection | KYB}
    title={t('title')}
    description={t('description')}
    image={getStepImage('STEP_ID')}
    progress={getOnboardingProgress(state.currentStep, state.stepStatus)}
    // ...variant-specific props
/>
```

Use `useOnboarding()` context for state: `updateFormData`, `completeAndNavigate`, `isSubmitting`, `state`.

---

## UI & Design System

### Design Tokens (from `@twin.org/ui-components-react` via TailwindConfig)

**Text colours:**
- `text-primary` — primary text (darkest)
- `text-secondary` — secondary text
- `text-tertiary` — tertiary/muted text
- `text-brand-primary` — orange brand colour (headings, links, active states)
- `text-brand-secondary` — blue brand colour (section headings)
- `text-surface-brand-secondary-1` — blue for table/section headings

**Background colours:**
- `bg-surface-main` — white (#FFFFFF)
- `bg-surface-second` — light grey (#F4F6F8) — use for page content areas
- `bg-surface-third` — medium grey (#DFE4EB)
- `bg-brand-primary` — orange brand

**Status/system colours:**
- `text-success` — green success
- `coral-400` / `coral-600` — danger/error borders and text
- `bg-system-error-tints-25` — light red background for error sections

**Spacing conventions:**
- Page top padding: `pt-16`
- Page horizontal padding: `px-6`
- Page max width: `max-w-6xl mx-auto`
- Section gap: `gap-y-8`
- Form field gap: `space-y-4` or `space-y-6`

### Component Library (`@twin.org/ui-components-react`)

**Always use these, never raw HTML equivalents:**

```tsx
// Buttons
import { Button, ButtonColors } from '@twin.org/ui-components-react';
<Button color={ButtonColors.Secondary}>Primary action</Button>
<Button color="plain">Secondary/cancel</Button>
<Button color="ghost" outline>Outlined</Button>

// Inputs — always sizing="lg" for forms
import { TextInput } from '@/components/ui-extension'; // project wrapper
<TextInput id="x" label="Label" sizing="lg" {...register('x')} requiredLabel />

// Select
import { Select } from '@twin.org/ui-components-react/select';
<Select options={[...]} sizing="lg" />

// Modal
import { Modal } from '@twin.org/ui-components-react';
<Modal size="lg" show={show} header="Title" onClose={onClose} body={...} footerButtons={[...]} />

// Tabs
import { Tabs, TabsVariants } from '@twin.org/ui-components-react';

// StepView (onboarding)
import { StepView, StepViewVariants } from '@twin.org/ui-components-react';
```

**Icons** — always from `@twin.org/ui-components-react/icons`:
```tsx
import { ArrowRight, Plus, Check, UserCircle, Lock } from '@twin.org/ui-components-react/icons';
// Always specify: type="bold"|"fill"|"light"|"regular", width, height
<ArrowRight type="bold" width={20} height={20} />
```

**PaginatedTable** — for all list/table views:
```tsx
import { PaginatedTable } from '@/components/paginated-table';
<PaginatedTable<MyType>
    columns={columns}           // ColumnDef<MyType>[]
    queryKey={['myData', id]}   // unique cache key
    queryFn={fetchFn}           // (cursor?, pageSize?) => Promise<ICursorPaginatedResponse<T>>
    placeholderData={serverData}
    getRowId={(row) => row.id}
    emptyMessage={tCommon('emptyState')}
/>
```

### Navigation — always locale-aware

```tsx
// ✅ Always use these
import { Link, useRouter, usePathname } from '@/i18n/navigation';

// ❌ Never use these (bypasses i18n)
import Link from 'next/link';
import { useRouter } from 'next/navigation';
```

---

## Translations (i18n)

**Rule: Every user-facing string must be in `en.json` AND `es.json`. Never hardcode UI text.**

**Files:** `app/messages/en.json` + `app/messages/es.json`

**Key naming convention:** `namespace.camelCase` — nested with dots
```json
{
    "actions": { "continue": "Continue", "cancel": "Cancel" },
    "pages": { "team": { "title": "{company}" } },
    "mySection": { "title": "Title", "subtitle": "Subtitle", "buttonLabel": "Click me" }
}
```

**Usage:**
```tsx
const t = useTranslations('mySection');
const tActions = useTranslations('actions');
t('title')          // "Title"
tActions('continue') // "Continue"

// Server components:
const t = await getTranslations('mySection');
```

**After adding keys:** run `npm run i18n-check` to validate.

**Standard shared keys** (use these, don't create duplicates):
- `actions.continue`, `actions.cancel`, `actions.saveChanges`, `actions.edit`, `actions.delete`, `actions.view`, `actions.add`
- `common.loadingText`, `common.processingButton`, `common.notSpecified`, `common.noDescription`
- `form.labels.*`, `form.errors.*`, `form.placeholders.*`

---

## Permissions (ABAC)

Permissions are fetched server-side in every protected `page.tsx`:

```tsx
const permResult = await database.getPermissionsForUser(organization.id);
const permissions = permResult.permissions; // PermissionMap: Record<string, Record<string, boolean>>

// Check permissions:
const canCreate = permissions?.template?.create ?? false;
const canApprove = permissions?.['credential-request']?.approve ?? false;
```

**Known permission resources & actions:**
- `template`: `list`, `create`, `edit`, `delete`
- `credential-request`: `approve`, `reject`, `view`
- `invitation`: `create`
- `agent`: `remove`
- `agent-role`: `update`
- `organization`: `settings`, `delete`

**Role hierarchy:** use `roleRank: RoleRankMap` (Record<string, number>) — higher number = higher privilege. Never hardcode role names.

**Client-side permissions:** use `usePermissions()` hook or `PermissionsContext` — never re-fetch on client if already passed as props.

---

## Forms

Always use `react-hook-form` + Zod:

```tsx
const schema = z.object({ email: z.string().email() });
type FormData = z.infer<typeof schema>;

const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema),
});
```

- Zod schemas live in `lib/schemas/` — one file per domain
- i18n error messages: pass `t` (translations) into schema factory functions
- Input error state: `color={errors.field ? 'failure' : ''}`
- Input helper text: `helperText={errors.field?.message}`
- All form inputs: `sizing="lg"`

---

## Data Fetching

**Server components:** use `database.*` methods directly (singleton: `lib/database/database.ts`)

**Client components:** use TanStack Query hooks from `hooks/`:
```tsx
const { data, isLoading, error } = useMyHook(id);
```

**Custom hook pattern:**
```tsx
// hooks/useMyData.ts
export function useMyData(id: string) {
    return useQuery({
        queryKey: ['myData', id],
        queryFn: async () => {
            const result = await apiFetch<{ data: MyType }>(`/api/my-data/${id}`);
            return result.data;
        },
        enabled: Boolean(id),
    });
}
```

**API calls from client:** always use `apiFetch` from `@/lib/utils/api-fetch` (handles errors consistently).

**Mutations:** use `useMutation` from TanStack Query, place in `hooks/` directory.

---

## API Routes

Pattern for all `app/api/*/route.ts`:
```tsx
import { NextResponse } from 'next/server';
import { z } from 'zod';
import { addCSRFHeaders } from '@/lib/utils/csrf-form'; // for POST requests

export async function POST(request: Request) {
    try {
        const body = await request.json();
        const validated = MySchema.parse(body);
        // ... process
        return NextResponse.json({ success: true }, { status: 201 });
    } catch (error) {
        if (error instanceof z.ZodError) {
            return NextResponse.json({ errors: error.errors }, { status: 400 });
        }
        logError(error, 'API: MyRoute error');
        return NextResponse.json({ error: 'Internal error' }, { status: 500 });
    }
}
```

**CSRF:** all mutating API calls (POST/PUT/DELETE) require CSRF headers via `addCSRFHeaders()`.

---

## Testing

### Unit Tests (Vitest)

- Location: `tests/unit/` — mirror source structure
- File naming: `[filename].test.ts` or `[filename].test.tsx`
- Run: `npm run test:unit`

```tsx
import { describe, it, expect, vi } from 'vitest';
import { myUtil } from '@/lib/utils/myUtil';

describe('myUtil', () => {
    it('does the expected thing', () => {
        expect(myUtil('input')).toBe('expected output');
    });
});
```

### E2E Tests (Playwright)

- Location: `e2e/tests/`
- Page objects: `e2e/page-objects/`
- Run: `npm run test:e2e`
- Always use Page Object Model pattern

```tsx
// e2e/page-objects/MyPage.ts
export class MyPage {
    constructor(private page: Page) {}
    async goto() { await this.page.goto('/en/my-page'); }
    async clickButton() { await this.page.getByTestId('my-button').click(); }
}

// e2e/tests/my-page.spec.ts
test('user can do the thing', async ({ page }) => {
    const myPage = new MyPage(page);
    await myPage.goto();
    await expect(page.getByTestId('my-button')).toBeVisible();
});
```

**data-testid:** add `data-testid` to all interactive elements and key content areas.

---

## Git & Branches

- Branch naming: `feature/`, `fix/`, `refactor/`, `test/`, `docs/`
- Commit messages: conventional commits — `feat:`, `fix:`, `refactor:`, `test:`, `docs:`
- **All commits must be GPG-signed** (repo rule enforced on push)
- Pre-commit: Husky runs lint-staged (ESLint + Prettier) automatically
- CI checks on PR: lint, typecheck, i18n-check

---

## Structural Changes Checklist

When making structural changes, update this file (`CLAUDE.md`) to reflect:
- [ ] New directories added
- [ ] New page patterns or layout changes
- [ ] New design tokens or component patterns
- [ ] New permissions/resources added
- [ ] New translation namespaces added
- [ ] New onboarding steps added

---

## Never Do

- Hardcode user-facing strings — always use i18n
- Import from `next/link` or `next/navigation` — use `@/i18n/navigation`
- Use `any` type — use `unknown` and narrow
- Use `surface-bg-first` — it doesn't exist; use `surface-second`
- Use `my-16` on the global layout `<main>` — use `pt-16` per page
- Add `px-24` inner indents on page headers — use `px-6` on the outer wrapper
- Hardcode role names — use `roleRank` for hierarchy checks
- Commit without running `typecheck` and `build`
- Push without GPG-signed commits
