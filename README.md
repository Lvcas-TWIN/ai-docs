# TWIN AI Docs

This repository is the single source of truth for AI agent documentation across all TWIN platforms.

## Purpose

When an AI coding agent (Claude, Cursor, Copilot, etc.) works in a TWIN codebase, it needs context to do its job correctly — what the stack is, what the conventions are, how the system fits together, what it's allowed to do, and how to run everything locally. Without this, agents make generic decisions that don't match the project's patterns, and developers spend time correcting mistakes instead of shipping.

This repository solves that by keeping all of that context in one place, organised by product team.

## Structure

```
ai-docs/
├── README.md                  ← you are here
└── <team-or-product>/
    ├── README.md              ← how the platform works and how to run it end-to-end
    ├── <repo-name>/
    │   ├── CLAUDE.md          ← project-specific AI rules (stack, patterns, conventions)
    │   ├── AGENTS.md          ← agent-oriented quick reference
    │   └── skills/            ← focused skill docs the agent loads on demand
    │       ├── <skill>.md
    │       └── ...
    └── ...
```

Each top-level folder belongs to a team or product area. Inside it, there is one subfolder per repository in that product. Every folder should contain enough documentation for an AI agent to:

- Understand what the repo does and how it fits into the larger system
- Run the entire platform locally (all services together)
- Run each repo in isolation for focused development
- Develop confidently in any part of the codebase

## What goes in each file

### `<team>/README.md`

The entry point for the whole platform. It should explain:

- What the product is and what problem it solves
- What each repository in the platform does and how they relate
- Links to the actual GitHub repositories
- **How to run the entire platform locally** — every service, in order, with env setup
- **How to run each repo in isolation** — for developers working on just one part
- How the repos connect at runtime (ports, env vars, API endpoints)

### `<repo>/CLAUDE.md`

Project-specific rules for AI agents working in that repository. This is the densest file — it should cover the full stack, all conventions, page/component patterns, design tokens, data fetching, forms, API routes, testing, git workflow, and a "never do" section. Think of it as the onboarding doc every new engineer and every AI agent reads first.

### `<repo>/AGENTS.md`

A quick-reference version of `CLAUDE.md` optimised for AI agents — commands, tech stack, project structure, code style, testing patterns, and hard boundaries (what to always do, what to ask first, what to never do). More concise than `CLAUDE.md`, structured for fast lookup.

### `<repo>/skills/`

Focused skill documents that an agent loads when working on a specific concern. Each skill covers one domain in depth — permissions logic, onboarding flows, UI patterns, testing standards, integration guides. Skills are narrow and detailed, whereas `CLAUDE.md` is broad and structural.

## How to use this repository

### For AI agents

When you start working in a TWIN codebase, locate the folder that matches your product area. Read the platform `README.md` first to understand the system, then read the specific repo's `CLAUDE.md` or `AGENTS.md` for development rules. Load the relevant skill docs when you're about to work on that domain (e.g. load `identity-onboarding.md` before touching any onboarding step).

### For developers

When you add a new repository, new conventions, or new architectural decisions to a TWIN product, update the corresponding files here. Stale docs produce bad agent output. Keep these files as up to date as you keep your actual code.

### For teams onboarding into TWIN

Create a new top-level folder for your team. Add a `README.md` that explains your platform end-to-end. Add one subfolder per repo with the relevant `CLAUDE.md`, `AGENTS.md`, and `skills/` docs.

## Current platforms

| Platform | Folder | Description |
|---|---|---|
| TWIN Identity | [`identity/`](./identity/) | Decentralised identity management — frontend, backend, and UI library |

## Adding a new platform

1. Create a folder at the top level: `mkdir <platform-name>/`
2. Add a `README.md` following the structure described above
3. Add one subfolder per repository in the platform
4. Populate each subfolder with the repo's AI docs
5. Add a row to the table above

The goal is that any developer or AI agent landing in this repo can find everything they need to work on any TWIN platform without needing to ask anyone.
