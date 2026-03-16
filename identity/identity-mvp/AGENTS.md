# AGENTS.md - AI Agent Guidelines for identity-mvp

> This document provides AI coding agents with the context they need to work effectively in this codebase.

---

## Commands

Run these commonly used commands from the project root (see `package.json` `scripts` for the authoritative and complete list):

```bash
# Development
npm run dev                    # Start Next.js dev server

# Build & Type Check
npm run build                  # Production build
npm run typecheck              # TypeScript type checking (tsc --noEmit)

# Testing
npm run test:unit              # Run Vitest unit tests
npm run test:unit:coverage     # Run unit tests with coverage
npm run test:e2e               # Run Playwright e2e tests

# Linting & Formatting
npm run lint                   # ESLint + Prettier check
npm run lint-fix               # Auto-fix ESLint issues
npm run format                 # Format with Prettier
npm run format:check           # Check formatting without writing

# i18n
npm run i18n-check             # Validate missing/invalid keys vs 'en'
```

---

## Tech Stack

- **Framework:** Next.js 16 with App Router (React 19, TypeScript 5)
- **Styling:** Tailwind CSS 3 + tailwind-merge + class-variance-authority
- **State/Data:** TanStack Query 5 (React Query), Zod for validation
- **Forms:** react-hook-form + @hookform/resolvers
- **UI Components:** Radix UI primitives, Flowbite React, Lucide icons, @twin.org/ui-components-react
- **i18n:** next-intl 4
- **Testing:** Vitest (unit), Playwright (e2e)
- **Linting:** ESLint 9 with TypeScript, React, and import plugins
- **Formatting:** Prettier

---

## Project Structure

```text
identity-mvp/
├── app/                      # Next.js App Router
│   ├── [locale]/             # Locale-based routing (i18n)
│   ├── api/                  # API routes
│   └── messages/             # i18n translation JSON files
├── components/               # Shared React components
│   └── ui/                   # Base UI components (buttons, inputs, etc.)
├── contexts/                 # React Context providers
├── hooks/                    # Custom React hooks
├── lib/                      # Core utilities
│   ├── schemas/              # Zod validation schemas
│   ├── interfaces/           # TypeScript interfaces
│   ├── services/             # API service functions
│   └── utils/                # Helper functions
├── providers/                # React providers (QueryClient, etc.)
├── e2e/                      # Playwright e2e tests
│   ├── page-objects/         # Page object models
│   └── tests/                # Test files
├── tests/                    # Unit tests (Vitest)
│   └── unit/                 # Unit test files
└── public/                   # Static assets
```

**Key patterns:**

- Route Groups: `(auth)`, `(protected)` for logical organization
- Client Components: `.client.tsx` suffix
- Server Components: `.tsx` (default in App Router)
- Path alias: `@/*` maps to project root

---

## Code Style

### Naming Conventions

```typescript
// Functions & variables: camelCase
const getUserData = () => {};
let isLoading = false;

// Components: PascalCase (function declaration required)
function UserProfile({ user }: UserProfileProps) {
  return <div>{user.name}</div>;
}

// Types & Interfaces: PascalCase
interface UserProfileProps {
  user: User;
}
type Status = 'pending' | 'approved' | 'rejected';

// Constants: UPPER_SNAKE_CASE (for true constants)
const API_BASE_URL = '/api/v1';
const MAX_RETRIES = 3;
```

### Component Example

```typescript
// ✅ Good - function declaration, proper types, hooks pattern
'use client';

import { useTranslations } from 'next-intl';
import { useQuery } from '@tanstack/react-query';

interface UserCardProps {
  userId: string;
  onSelect?: (id: string) => void;
}

function UserCard({ userId, onSelect }: UserCardProps) {
  const t = useTranslations('profile');
  
  const { data, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <div className="rounded-lg border p-4">
      <h2>{data.name}</h2>
      <p>{t('greeting', { name: data.name })}</p>
      {onSelect && (
        <button type="button" onClick={() => onSelect(userId)}>
          {t('select')}
        </button>
      )}
    </div>
  );
}

export default UserCard;
```

### TypeScript Requirements

```typescript
// ✅ Use type imports
import { type User } from '@/lib/interfaces/user';

// ✅ Prefer inline type imports when mixed
import { fetchUser, type UserResponse } from '@/lib/services/user';

// ✅ Mark unused variables with underscore prefix
const handleSubmit = (_event: FormEvent) => {};

// ❌ Never use `any` - use `unknown` and narrow
```

### i18n - Always Use Translations

```typescript
// ✅ Good - use translations for all user-facing text
const t = useTranslations('login');
return <h1>{t('title')}</h1>;

// ❌ Bad - hardcoded strings (lint error: react/jsx-no-literals)
return <h1>Login to your account</h1>;
```

### Navigation - Use i18n-aware imports

```typescript
// ✅ Good - locale-aware navigation
import { Link, useRouter, usePathname } from '@/i18n/navigation';

// ❌ Bad - these bypass i18n (lint error)
import Link from 'next/link';
import { useRouter } from 'next/navigation';
```

---

## Testing Standards

### Unit Tests (Vitest)

Location: `tests/unit/` - mirror the source structure

```typescript
// tests/unit/lib/utils/formatDate.test.ts
import { describe, it, expect } from 'vitest';
import { formatDate } from '@/lib/utils/formatDate';

describe('formatDate', () => {
  it('formats ISO date to readable string', () => {
    expect(formatDate('2024-01-15')).toBe('January 15, 2024');
  });

  it('handles invalid input gracefully', () => {
    expect(formatDate('')).toBe('Invalid date');
  });
});
```

### E2E Tests (Playwright)

Location: `e2e/tests/` - use page objects

```typescript
// e2e/tests/login.spec.ts
import { test, expect } from '../fixtures';
import { LoginPage } from '../page-objects/LoginPage';

test.describe('Login Flow', () => {
  test('user can log in with valid credentials', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('user@example.com', 'password');
    await expect(page).toHaveURL(/\/dashboard/);
  });
});
```

---

## Git Workflow

1. **Before committing:** Husky runs lint-staged automatically
2. **Commit messages:** Use conventional commits (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`)
3. **Pre-commit checks:** ESLint + Prettier run on staged files

---

## Boundaries

### ✅ Always Do

- Run `npm run lint` before considering work complete
- Run `npm run typecheck` to verify TypeScript compiles
- Use translations for ALL user-facing strings (update all locale files in `app/messages/`, currently `app/messages/en.json` and `app/messages/es.json`)
- Use `@/` path alias for imports
- Write tests for new utility functions
- Use TanStack Query for server state management
- Use Zod schemas in `lib/schemas/` for validation

### ⚠️ Ask First

- Adding new npm dependencies
- Modifying database schemas or API contracts
- Changes to `next.config.ts` or `eslint.config.mjs`
- Modifying CI/CD configuration in `.github/`
- Architectural changes to routing or state management
- Removing or renaming existing translations

### 🚫 Never Do

- Commit secrets, API keys, or credentials (use environment variables)
- Modify `node_modules/` or lockfiles (`package-lock.json`) directly
- Use `any` type without explicit justification
- Hardcode user-facing strings (use i18n)
- Import directly from `next/link` or `next/navigation` (use `@/i18n/navigation`)
- Delete failing tests without fixing the underlying issue
- Modify `.env*` files with real credentials
- Push directly to main branch

---

## Environment Variables

Required variables are documented in `.env.example`. Never commit real values.

```bash
# Reference only - actual values in .env.local
NEXT_PUBLIC_API_URL=
SENTRY_DSN=
UPSTASH_REDIS_URL=
NEXT_IDENTITY_MANAGEMENT_URL=
```

---

## Common Patterns

### API Route Handler

```typescript
// app/api/users/route.ts
import { NextResponse } from 'next/server';
import { z } from 'zod';

const UserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
});

export async function POST(request: Request) {
  try {
    const body = await request.json();
    const validated = UserSchema.parse(body);
    
    // Process validated data...
    
    return NextResponse.json({ success: true }, { status: 201 });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json({ errors: error.errors }, { status: 400 });
    }
    return NextResponse.json({ error: 'Internal error' }, { status: 500 });
  }
}
```

### Custom Hook with TanStack Query

```typescript
// hooks/useUser.ts
import { useQuery } from '@tanstack/react-query';
import { type User } from '@/lib/interfaces/user';

export function useUser(userId: string) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: async (): Promise<User> => {
      const res = await fetch(`/api/users/${userId}`);
      if (!res.ok) throw new Error('Failed to fetch user');
      return res.json();
    },
    enabled: Boolean(userId),
  });
}
```

### Form with react-hook-form + Zod

```typescript
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const formSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

type FormData = z.infer<typeof formSchema>;

function LoginForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(formSchema),
  });

  const onSubmit = (data: FormData) => {
    // Handle submission
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}
      {/* ... */}
    </form>
  );
}
```

---

## Quick Reference

| Task | Command |
| --- | --- |
| Start dev server | `npm run dev` |
| Run all checks | `npm run lint && npm run typecheck && npm run test:unit` |
| Format code | `npm run format` |
| Run e2e tests | `npm run test:e2e` |
| Check translations | `npm run i18n-check` |
