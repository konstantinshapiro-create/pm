# Code Review — Full Codebase

**Reviewer:** Claude (Sonnet 5)
**Date:** 2026-09-19
**Scope:** Comprehensive review of the entire current codebase. There is no
backend or scripts code yet (`backend/` and `scripts/` each contain only a
placeholder `AGENTS.md`), so in practice this covers all of `frontend/` — the
standalone, in-memory Kanban demo described in `CLAUDE.md`.

**Method:** Read every source, test, and config file in full; cross-checked
`moveCard` drag/drop logic against `@dnd-kit/sortable`'s own `arrayMove`
semantics; ran `npm run lint`, `npm run test:unit`, `npm run build`, and
`npx tsc --noEmit` to verify claims against actual tool output rather than
inspection alone; diffed against the working tree's uncommitted changes and
the pre-existing `docs/code-review.md` to confirm which previously reported
issues are still live.

**Files read in full:** `src/lib/kanban.ts`, `src/lib/kanban.test.ts`,
`src/components/KanbanBoard.tsx`, `KanbanBoard.test.tsx`, `KanbanCard.tsx`,
`KanbanCardPreview.tsx`, `KanbanColumn.tsx`, `NewCardForm.tsx`,
`src/app/page.tsx`, `src/app/layout.tsx`, `src/app/globals.css`,
`src/test/setup.ts`, `src/test/vitest.d.ts`, `tests/kanban.spec.ts`,
`package.json`, `vitest.config.ts`, `playwright.config.ts`, `tsconfig.json`,
`next.config.ts`, `eslint.config.mjs`, `.gitignore`, plus repo-root
`AGENTS.md`, `CLAUDE.md`, `docs/PLAN.md`, `docs/code-review.md`.

## Summary

The codebase is small, clean, and matches the "keep it simple" standard set
in `AGENTS.md`. Lint is clean, all 6 unit tests pass, and `next build`
succeeds. No crash-level correctness bugs were found in the core `moveCard`
drag-and-drop logic — it was checked branch-by-branch (same-column reorder,
cross-column move, drop-on-column vs. drop-on-card) and matches dnd-kit's own
`arrayMove` semantics.

One issue from the earlier `docs/code-review.md` review (missing keyboard
sensor for drag-and-drop) has since been fixed in the working tree — a
`KeyboardSensor` was added and covered with a Playwright test. The remaining
findings below are new or still outstanding.

## Findings

### 1. [Medium] Card is missing any edit capability, despite being a stated requirement

**File:** `frontend/src/components/KanbanCard.tsx`, `frontend/src/components/NewCardForm.tsx`

`AGENTS.md` states: *"The cards on the Kanban board can be moved with drag
and drop, and **edited**."* The current implementation only supports
creating a card (`NewCardForm`) and deleting it (`KanbanCard`'s "Remove"
button) — there is no way to change a card's title or details after it has
been created. A user who makes a typo in a card's details, or whose scope
changes, must delete the card and re-create it from scratch, losing any
context about when it was originally added. This is a functional gap
against the documented requirements rather than a bug in existing code, but
worth flagging explicitly before this is mistaken for "done" — `docs/PLAN.md`
is still an unexpanded outline, so there is no checklist item to catch this
otherwise.

**Suggested fix:** add an edit affordance (e.g. click-to-edit on the card
title/details, or an "Edit" button opening the same form used for creation)
before considering the Kanban card feature complete.

### 2. [Medium] `<button>` nested inside a dnd-kit drag handle produces invalid interactive-in-interactive markup

**File:** `frontend/src/components/KanbanCard.tsx:21-31`

`attributes` and `listeners` from `useSortable` are spread onto the whole
`<article>`, which gives it dnd-kit's default `role="button" tabIndex={0}`.
A real `<button aria-label="Delete ...">` is then nested inside that
`role="button"` element:

```tsx
<article ref={setNodeRef} style={style} {...attributes} {...listeners} ...>
  ...
  <button onClick={() => onDelete(card.id)} aria-label={`Delete ${card.title}`}>
    Remove
  </button>
</article>
```

Interactive elements nested inside other interactive elements are invalid
per the HTML/ARIA spec and are mis-announced by screen readers (nested
"button" roles are either collapsed or produce confusing double
announcements). Tabbing through the board also lands first on the outer
`role="button"` card, which does nothing on Enter/Space by itself — the user
has to tab a second time to reach the real "Remove" button.

**Suggested fix:** move `{...attributes} {...listeners}` onto a dedicated
drag handle element (e.g. a small grip icon) instead of the whole card, so
the card's semantic content and the "Remove" button aren't nested inside a
synthetic button role.

### 3. [Low] `vitest.d.ts` references the wrong module, so `tsc --noEmit` fails on every test file

**File:** `frontend/src/test/vitest.d.ts:1`

```ts
/// <reference types="vitest" />
```

`vitest.config.ts` sets `globals: true`, so tests call `describe`/`it`/
`expect` as ambient globals. Vitest ships the type declarations for those
globals in a separate `vitest/globals` entry point
(`node_modules/vitest/globals.d.ts`), not in the base `vitest` package. The
current reference pulls in vitest's regular (non-global) types, which don't
declare these globals. Running `npx tsc --noEmit` (a standard sanity check,
and what most editors' TypeScript language servers do under the hood)
confirms this — it fails with 16 errors, e.g.:

```
src/lib/kanban.test.ts(3,1): error TS2582: Cannot find name 'describe'.
src/lib/kanban.test.ts(11,5): error TS2304: Cannot find name 'expect'.
```

`npm run test:unit` (vitest itself) and `npm run build` (`next build`, which
only type-checks files reachable from the app entry points, not the
standalone test files) both still pass, which is likely why this has gone
unnoticed — but any contributor running `tsc --noEmit` directly, or relying
on their editor's inline TypeScript diagnostics while writing tests, will
see spurious red squiggles on every `describe`/`it`/`expect` call.

**Suggested fix:** change the reference to
`/// <reference types="vitest/globals" />` (or add `"vitest/globals"` to
`tsconfig.json`'s `compilerOptions.types`).

### 4. [Low] Same-column reorder logic re-implements `arrayMove`, which the project already depends on

**File:** `frontend/src/lib/kanban.ts:125-127`

```ts
const nextCardIds = [...activeColumn.cardIds];
nextCardIds.splice(oldIndex, 1);
nextCardIds.splice(newIndex, 0, activeId);
```

`@dnd-kit/sortable` — already a direct dependency — exports `arrayMove(array,
from, to)`, which does exactly this remove-then-reinsert. The current logic
is correct (verified against `arrayMove`'s own behavior for every branch),
but any future edge-case fix (e.g. an off-by-one at list boundaries) has to
be re-derived and re-tested here instead of relying on the library's
well-tested implementation, and invites a future contributor to "fix" a
perceived bug that upstream already handles.

**Suggested fix:** replace the manual splice pair with
`arrayMove(activeColumn.cardIds, oldIndex, newIndex)`.

### 5. [Low] Playwright's `test-results/` output is not gitignored

**File:** `frontend/.gitignore`

Only `/coverage` is ignored under the "testing" section; `test-results/` (Playwright's report/trace output directory, configured with
`trace: "retain-on-failure"` in `playwright.config.ts`) is not. This is
directly observable right now: `git status` shows
`frontend/test-results/.last-run.json` as modified, purely from having run
`npm run test:e2e` — no code change involved. If any e2e test ever fails
locally, this will start writing trace `.zip` files and screenshots into a
tracked directory, and the next `git add -A`/`git add .` will commit binary
test artifacts into version control.

**Suggested fix:** add `/test-results/` (and `/playwright-report/` if that
reporter is ever enabled) to `frontend/.gitignore`, and remove
`test-results/.last-run.json` from tracking.

### 6. [Nit] No-op `useMemo` around `board.cards`

**File:** `frontend/src/components/KanbanBoard.tsx:33`

```ts
const cardsById = useMemo(() => board.cards, [board.cards]);
```

This computes nothing — it returns the same reference already in scope. It
costs a dependency-array comparison on every render for no benefit.

**Suggested fix:** `const cardsById = board.cards;`

### 7. [Nit] No focus management when opening the "add card" form

**File:** `frontend/src/components/NewCardForm.tsx`

Clicking "Add a card" swaps in the title/details form but does not move
focus into the new title input (no `autoFocus`, no `ref().focus()`).
Keyboard and screen-reader users have to locate and re-focus the newly
revealed field themselves. Small, but easy to fix alongside finding #2 while
touching card-related accessibility.

**Suggested fix:** add `autoFocus` to the title `<input>`, or focus it
imperatively in the `onClick` handler that sets `isOpen`.

## Verified as fixed (from prior review)

- **Keyboard-inaccessible drag-and-drop** — `KanbanBoard.tsx` now registers
  a `KeyboardSensor` with `sortableKeyboardCoordinates` alongside the
  `PointerSensor`, and `tests/kanban.spec.ts` has a new passing e2e test
  ("reorders a card within a column via keyboard") exercising it. Confirmed
  by reading the current source and running the test suite.

## What's already good

- `moveCard` (the trickiest piece of logic in the codebase) is pure,
  well-factored, and has correct behavior for every branch checked against
  `arrayMove` semantics.
- Data model (`BoardData` normalized by id, columns referencing cards by id
  only) is a sound choice that will make backend/API integration
  straightforward later.
- Lint is clean, all unit tests pass, and the production build succeeds.
- No hardcoded colors outside the CSS custom-property tokens — matches the
  `AGENTS.md` color-scheme convention.
- No over-engineering: no premature state-management library, no speculative
  abstractions ahead of the backend that doesn't exist yet.
