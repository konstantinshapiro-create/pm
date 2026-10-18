2# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

This is a Project Management MVP: a single-user Kanban board with an AI chat sidebar that can
create/edit/move cards. Full business requirements, technical decisions, and color scheme are in
`AGENTS.md` at the repo root — read it before making architectural decisions. The detailed,
checklist-style build plan (with phases and success criteria) is in `docs/PLAN.md` — check it for
which phase of the project is currently in progress before assuming a feature exists.

Target architecture (see AGENTS.md for full detail): NextJS frontend + Python FastAPI backend
(serving the built static NextJS site at `/`), SQLite database, OpenRouter for AI calls
(`openai/gpt-oss-120b`), everything packaged in a single Docker container, with `uv` as the Python
package manager. Hardcoded MVP login is `user`/`password`; only one Kanban board per user.

**Current state:** only `frontend/` is implemented, as a standalone frontend-only demo (no backend,
no auth, no persistence, no AI wiring yet — the board resets to in-memory seed data on reload).
`backend/` and `scripts/` are placeholders (each contains only a stub `AGENTS.md`). Do not assume
backend routes, a database, or Docker setup exist until they've actually been built.

## Commands

All frontend commands run from `frontend/`:

```bash
npm install
npm run dev          # start Next.js dev server
npm run build
npm run lint         # eslint
npm run test:unit         # vitest run (unit tests, jsdom)
npm run test:unit:watch   # vitest watch mode
npm run test:e2e          # playwright (auto-starts dev server on 127.0.0.1:3000)
npm run test:all          # unit + e2e
```

Run a single unit test file: `npx vitest run src/components/KanbanBoard.test.tsx`
Run a single e2e test: `npx playwright test tests/kanban.spec.ts`

There is no backend yet, so there are no backend build/test/lint commands to run.

## Frontend architecture

State lives entirely client-side in `KanbanBoard.tsx` (a single `"use client"` component holding
the whole `BoardData` in `useState`, with no external state library). Data flows one-directional:
`KanbanBoard` owns state and handler callbacks, passes them down to `KanbanColumn` → `KanbanCard`.

Core data model and pure logic (`src/lib/kanban.ts`):
- `BoardData` = `{ columns: Column[], cards: Record<string, Card> }` — cards are stored normalized
  by id, columns reference cards only by id (`cardIds: string[]`).
- `moveCard(columns, activeId, overId)` is the pure function implementing drag-and-drop reordering
  logic (same-column reorder, cross-column move, drop-on-column vs drop-on-card) — this is the
  most subtle piece of logic in the codebase and is covered by `kanban.test.ts`.
- `createId(prefix)` generates client-side ids for new cards; this will need to be reconciled with
  server-assigned ids once the backend exists.

Drag-and-drop uses `@dnd-kit/core` (`DndContext`, `closestCorners` collision detection,
`DragOverlay` for the drag preview via `KanbanCardPreview`). When touching drag/drop behavior,
changes usually span `KanbanBoard.tsx` (event handlers), `lib/kanban.ts` (the pure move logic), and
`KanbanColumn`/`KanbanCard` (sortable item wiring) together.

Styling is Tailwind v4 (via `@tailwindcss/postcss`) using CSS custom properties for the brand
palette (see AGENTS.md Color Scheme) rather than hardcoded hex values in components — reuse the
existing `var(--...)` tokens (e.g. `--accent-yellow`, `--primary-blue`, `--navy-dark`, `--gray-text`)
rather than introducing new colors.

Path alias `@/*` maps to `frontend/src/*` (configured in both `tsconfig.json` and
`vitest.config.ts` — keep them in sync if it changes).

Testing setup: Vitest + Testing Library + jsdom for unit tests (colocated `*.test.ts(x)` files
under `src/`), Playwright for e2e (`tests/*.spec.ts`, against a real dev server on port 3000).

## Coding standards (from AGENTS.md)

- Use latest/idiomatic versions of libraries as of today.
- Keep it simple — no over-engineering, no unnecessary defensive programming, no speculative
  features.
- No emojis, ever.
- When debugging, find the root cause before attempting a fix — don't guess-and-check.
