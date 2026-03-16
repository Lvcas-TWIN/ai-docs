---
name: identity-development
description: Development rules and patterns for the identity-mvp project. Use whenever writing, editing or reviewing code in this codebase.
---

# identity-mvp Development Guide

## Before You Start
- Read `CLAUDE.md` at the repo root — it is the source of truth for this project
- Run `npm run typecheck` after every change
- Run `npm run build` before every push

## Stack
- Next.js 16 + React 19 + TypeScript 5
- Tailwind CSS 3 with `@twin.org/ui-components-react` tokens
- TanStack Query 5 for server state
- react-hook-form + Zod for forms
- next-intl 4 for i18n

## Page Patterns

### Protected page (server + client split)
```
app/[locale]/(protected)/[section]/page.tsx         ← server: data + permissions
app/[locale]/(protected)/[section]/[Name]PageClient.tsx  ← client: UI
```

Server `page.tsx` always:
1. Calls `getAuthenticatedAgent()`
2. Calls `database.getPermissionsForUser(organization.id)`
3. Passes data + permission booleans as props to client component

Client `PageClient.tsx` layout (MUST match Team page — canonical reference):
```tsx
<div className="max-w-6xl mx-auto px-6 flex flex-col gap-y-8 pt-16">
    <div className="flex flex-col gap-y-2">
        <h1 className="text-3xl font-bold text-brand-primary">{t('title')}</h1>
        <p className="text-base text-tertiary">{t('description')}</p>
    </div>
    {/* section heading + add button */}
    <div className="flex justify-between items-end mb-2">
        <h3 className="text-lg font-semibold text-surface-brand-secondary-1">{t('section')}</h3>
        <Button color={ButtonColors.Secondary}>+ Add</Button>
    </div>
    <PaginatedTable<T> ... />
</div>
```

### Auth page
All auth pages use `AuthLayout` from `@/components/auth-layout/AuthLayout`. Split layout: 30% image / 70% content.

### Onboarding step
All onboarding steps use `StepView` from `@twin.org/ui-components-react` + `useOnboarding()` context.

## Mandatory Rules

### Navigation — always locale-aware
```tsx
// ✅
import { Link, useRouter, usePathname } from '@/i18n/navigation';
// ❌ Never
import Link from 'next/link';
import { useRouter } from 'next/navigation';
```

### i18n — never hardcode UI strings
```tsx
const t = useTranslations('mySection');
// Both en.json AND es.json must be updated together
```

### Forms — always react-hook-form + Zod + sizing="lg"
```tsx
const { register, handleSubmit, formState: { errors } } = useForm<T>({
    resolver: zodResolver(schema),
});
<TextInput sizing="lg" color={errors.field ? 'failure' : ''} helperText={errors.field?.message} />
```

### Data fetching
- Server components → `database.*` directly
- Client components → TanStack Query hooks in `hooks/`
- API calls → `apiFetch` from `@/lib/utils/api-fetch`

### TypeScript
- Never use `any` — use `unknown` and narrow
- Use `type` imports: `import { type MyType } from '...'`

## Tokens That Do NOT Exist
- `surface-bg-first` → use `bg-surface-second` instead

## Checklist Before Submitting
- [ ] `npm run typecheck` passes
- [ ] `npm run build` passes
- [ ] Both `en.json` and `es.json` updated if strings changed
- [ ] `npm run i18n-check` passes
- [ ] `data-testid` added to new interactive elements
- [ ] No hardcoded UI strings
- [ ] No imports from `next/link` or `next/navigation`
