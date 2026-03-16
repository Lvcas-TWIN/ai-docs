# TWIN Identity Platform

TWIN ID is a compliant framework for creating, managing, and verifying digital identities across TWIN trade ecosystems. It uses decentralised identity (DID) and verifiable credentials (VCs) to enable secure, interoperable trust between organisations.

## Repositories

| Repo | GitHub | Description |
|---|---|---|
| `identity-mvp` | https://github.com/twinfoundation/identity-mvp | Frontend — Next.js app, the user-facing product |
| `identity-management` | https://github.com/twinfoundation/identity-management | Backend — REST API, business logic, credential workflows |
| `ui` | https://github.com/twinfoundation/ui | Shared component library — React + Svelte, design tokens |

## How the pieces connect

```
Browser
  └── identity-mvp (Next.js, :3000)
        ├── calls REST API → identity-management (:3001)
        └── renders components from @twin.org/ui-components-react (built from ui/)
```

- The **frontend** (`identity-mvp`) is the only user-facing surface. It handles auth, onboarding, credential management, team management, and organisation settings.
- The **backend** (`identity-management`) owns all business logic: DID creation, credential issuance, ABAC permissions, onboarding state, invitations, and the REST API.
- The **UI library** (`ui`) is a build-time dependency of `identity-mvp`. You don't run it as a service — you build it locally and link it when developing components.

---

## Running the full platform locally

This is the recommended setup for working across the whole system at once.

### Prerequisites

- Node.js >= 22.x (identity-mvp requires 22.x; identity-management requires >= 20.x)
- npm
- Git

### Step 1 — Clone all three repos

```bash
git clone https://github.com/twinfoundation/identity-mvp.git
git clone https://github.com/twinfoundation/identity-management.git
git clone https://github.com/twinfoundation/ui.git
```

### Step 2 — Set up and start the backend (`identity-management`)

```bash
cd identity-management

# Install dependencies
npm install

# Build all packages (includes tests)
npm run dist

# OR faster build without tests
npm run dist:no-test

# Set up environment variables
cp apps/identity-management-node/.env.example apps/identity-management-node/.env
# Edit .env with your local configuration (see Configuration section below)

# Start in development mode (watches for changes, auto-restarts)
cd apps/identity-management-node
npm run dev
```

The backend will be available at **http://localhost:3001**.
OpenAPI spec is at **http://localhost:3001/spec**.

> **Production mode:** after a full build, run `npm start` from `apps/identity-management-node/` instead of `npm run dev`.

### Step 3 — Set up and start the frontend (`identity-mvp`)

Open a new terminal:

```bash
cd identity-mvp

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env.local
# Edit .env.local — make sure NEXT_IDENTITY_MANAGEMENT_URL=http://localhost:3001

# Start the dev server
npm run dev
```

The frontend will be available at **http://localhost:3000**.

### Step 4 — Verify everything is connected

1. Open http://localhost:3000
2. Try registering a new organisation — this will trigger the full onboarding flow, calling identity-management at :3001
3. Check http://localhost:3001/spec to confirm the backend API is reachable

### Environment variables — what matters most

**identity-management** (`apps/identity-management-node/.env`):
```bash
IDENTITY_PORT=3001                    # port the service runs on
# Additional IDENTITY_* variables for vault, storage, etc — see docs/configuration.md
```

**identity-mvp** (`.env.local`):
```bash
NEXT_IDENTITY_MANAGEMENT_URL=http://localhost:3001   # points to backend
UPSTASH_REDIS_URL=...                                 # Redis for session storage
SENTRY_DSN=...                                        # optional, error monitoring
```

---

## Running each repo in isolation

### `identity-management` — backend only

Use this when working on API endpoints, business logic, permissions, or credential workflows without needing the frontend.

```bash
cd identity-management
npm install
npm run dist:no-test          # build everything
cd apps/identity-management-node
npm run dev                   # watch mode — auto-rebuilds and restarts
```

**Useful commands from repo root:**
```bash
npm run dist          # full build with tests
npm run dist:no-test  # fast build, skip tests
npm run lint          # ESLint + Prettier + markdownlint + cspell
npm run docs          # generate OpenAPI spec
```

**Useful commands from `apps/identity-management-node/`:**
```bash
npm run dev           # development mode (watch + nodemon)
npm start             # production mode (runs compiled dist/)
npm test              # run Vitest tests
npm run test:coverage # run tests with coverage
```

**Test email templates:**
```bash
cd packages/identity-management-service
npm run email-dev     # starts local preview server for React Email templates
```

**Running with Docker:**
```bash
cp .env.example .env
docker buildx build . --tag identity-management --file apps/identity-management-node/deploy/Dockerfile
docker run --tty --interactive --rm --env-file .env identity-management
# Available at http://localhost:3001
```

---

### `identity-mvp` — frontend only

Use this when working on UI, pages, onboarding steps, or anything that doesn't require backend changes. Point it at the hosted staging backend if you don't want to run identity-management locally.

```bash
cd identity-mvp
npm install
cp .env.example .env.local
# Set NEXT_IDENTITY_MANAGEMENT_URL to either http://localhost:3001 or a staging URL
npm run dev
# Available at http://localhost:3000
```

**Key commands:**
```bash
npm run dev              # start dev server
npm run build            # production build — run before every push
npm run typecheck        # TypeScript type check — run after every change
npm run lint             # ESLint + Prettier
npm run lint-fix         # auto-fix lint issues
npm run format           # Prettier write
npm run test:unit        # Vitest unit tests
npm run test:unit:coverage
npm run test:e2e         # Playwright end-to-end tests
npm run i18n-check       # validate translation keys vs en.json
```

**Running e2e tests:**
```bash
# install browsers first (once)
npx playwright install

# run tests (dev server must be running)
npm run test:e2e

# interactive UI mode
npm run test:e2e -- --ui

# generate test code by recording interactions
npx playwright codegen http://localhost:3000
```

> Note: Playwright also requires a Redis connection — it runs `e2e/setup/global-setup.ts` to reset the database before tests.

---

### `ui` — component library only

Use this when developing or updating UI components, design tokens, or Storybook stories. This is a build-time dependency — you don't run it as a service.

```bash
cd ui
npm install

# Build the Tailwind design token package first (always required)
cd packages/ui-tailwind
npm run dist
cd ../..

# Develop React components with hot reload
cd packages/ui-components-react
npm run dev
# (keep this terminal running)

# Run React Storybook (in a separate terminal)
cd apps/ui-storybook-react
npm run storybook
# Available at http://localhost:6006
```

**For Svelte components:**
```bash
cd packages/ui-components-svelte
npm run dev

# Svelte Storybook (separate terminal)
cd apps/ui-storybook-svelte
npm run storybook
```

**Key commands from repo root:**
```bash
npm run build          # TypeScript compilation — must be clean before push
npm run test           # all tests across all packages
npm run lint           # ESLint + Prettier + markdownlint + cspell
```

**To link the local UI build into identity-mvp** (for testing component changes end-to-end):
```bash
# From ui/
npm run local-link

# Then in identity-mvp, install with local linked packages
npm install
```

---

## AI documentation in this folder

Each subfolder contains the Claude/AI agent docs for that repository:

```
identity/
├── README.md                              ← this file
├── identity-mvp/
│   ├── CLAUDE.md                          ← full dev rules and patterns
│   ├── AGENTS.md                          ← quick reference for agents
│   └── skills/
│       ├── identity-business-logic.md     ← ABAC, permissions, credential workflow
│       ├── identity-development.md        ← dev patterns and mandatory rules
│       ├── identity-onboarding.md         ← full onboarding state machine guide
│       ├── identity-product-design.md     ← UI tokens, component patterns, layout
│       └── identity-qa.md                 ← testing standards (Vitest + Playwright)
├── identity-management/
│   └── skills/
│       └── identity-new-service.md        ← integrating a new external team/service
└── ui/
    ├── CLAUDE.md                          ← component authoring rules, monorepo layout
    └── skills/
        ├── twin-ui-business-logic.md      ← component behavior contracts, publishing
        ├── twin-ui-development.md         ← dev rules, file structure, code style
        ├── twin-ui-new-component.md       ← end-to-end guide for adding a new component
        ├── twin-ui-product-design.md      ← design tokens, variants, visual rules
        ├── twin-ui-qa.md                  ← testing standards for components
        └── twin-ui-storybook.md           ← writing and maintaining Storybook stories
```
