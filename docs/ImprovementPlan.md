# Improvement Plan: `app/`

**Date:** 2026-10-28
**Inputs:** [ReviewFindings.md](ReviewFindings.md), the kata's smell list in [app/README.md](../app/README.md)

## Goals

1. Refactor away the code smells **without changing behaviour**, in small steps, keeping the tests green after every step (the kata's rule).
2. Then fix the four validated bugs with test-first changes.
3. Leave `npm run build`, `npm test` and `npm run lint` all passing.

## Constraints

- Keep the public API the tests use: `new Game()`, `Play(symbol, x, y)`, which throws on an invalid move, and `Winner()`, which returns `'X'`, `'O'` or `' '`. Internals may change freely, but any change to the API is a separate, explicit decision made in phase 4.
- Refactor in phase 2 and change behaviour in phase 3. Never do both in one commit.
- Commit after each green step, so any step can be reverted on its own.

---

## Phase 0: Working baseline

| # | Step | Done when |
|---|------|-----------|
| 0.1 | `npm install` in `app/` | `node_modules` present |
| 0.2 | Run `npm test` | All 10 tests pass (recorded as the baseline) |
| 0.3 | Run `npm run lint` and record the errors | List of `eqeqeq` errors (Game.ts) and `no-var` errors (Game.test.ts) |
| 0.4 | Fix `tsconfig.json` `outDir` (`./dist`) to match `package.json` (`main: ./build/index.js`, `clean: rm -rf build`): set `outDir: ./build` | `npm run build` writes to `build/` |

## Phase 1: Safety net

These tests pin down **current** behaviour so refactoring can't change it unnoticed.

| # | Step |
|---|------|
| 1.1 | Add one characterization test that `Winner()` returns `' '` for an empty board and for a board with no full row |
| 1.2 | Add a test that a full row of mixed symbols isn't a win |
| 1.3 | Replace `var` with `const` in `Game.test.ts`, which clears the `no-var` lint errors |

No tests for columns, diagonals or invalid input yet. They would assert the buggy behaviour and then have to be reversed in phase 3.

## Phase 2: Refactor the smells, behaviour unchanged

Ordered so each step makes the next one easier. Run `npm test` after every step.

| # | Smell | Change |
|---|-------|--------|
| 2.1 | Comments | Delete comments that restate the code, plus the wrong ones (`//if the positions in first row are taken` repeated for every row, `//if player is X` above a check for O) |
| 2.2 | Lint / `eqeqeq` | Replace `==` / `!=` with `===` / `!==` throughout `Game.ts` |
| 2.3 | Dead code | Remove the unused `tile` variable in `Board.AddTileAt` |
| 2.4 | Primitive obsession | Introduce constants or a type for symbols: `type Player = 'X' \| 'O'`, `const EMPTY = ' '`. Use them everywhere `'X'`, `'O'` and `' '` appear as literals |
| 2.5 | Data clump / long parameter list | Introduce a `Position` (`{ x, y }`) value used by `Board` internally, so `x, y` stop travelling as separate arguments. `Game.Play`'s public signature stays the same |
| 2.6 | Duplicated code / long method | Replace the three copied row blocks in `Winner()` with a list of winning lines that is checked in one loop (`lines.find(isComplete)`). For now the list contains **only the three rows**, so behaviour is unchanged |
| 2.7 | Message chain / feature envy | Move "is this line won, and by whom" from `Game` into `Board` (e.g. `board.winnerOf(line)`), so `Game` stops reaching through `TileAt(...).Symbol` |
| 2.8 | Data class | Give `Tile` behaviour (`isEmpty()`, `isTakenBy(player)`) or replace it with a 3×3 array inside `Board`. Remove the `find(...)!` non-null assertions |
| 2.9 | Large class / divergent change | Split `Game` into separate responsibilities: turn rules (who may move) and outcome (who has won). Each reason to change then touches one place |
| 2.10 | Lazy class / shotgun surgery | Review after 2.9: merge any class left with no real job, and check that adding a rule touches one file only |

**Checkpoint:** all original tests plus phase 1 tests pass, `npm run lint` is clean, and the diff contains no behaviour change.

## Phase 3: Fix the validated bugs (test-first)

For each bug: write a failing test, make the smallest fix, then commit. References are to [ReviewFindings.md](ReviewFindings.md).

| # | Bug | Failing test to add first | Fix |
|---|-----|---------------------------|-----|
| 3.1 | #1 Winner checks rows only | Column win for X and O; both diagonals | Add 3 columns and 2 diagonals to the winning-lines list from 2.6. This is one line of data |
| 3.2 | #2 Out-of-range coordinates | `Play('X', 3, 3)` throws "Invalid position", and the game state is unchanged afterwards (a following `Play('X', 0, 0)` succeeds) | Validate bounds **before** any state update |
| 3.3 | #3 Any symbol accepted | `Play('Z', 0, 0)` and `Play(' ', 0, 0)` throw | Validate the symbol is `'X'` or `'O'` (falls out naturally from the `Player` type in 2.4) |
| 3.4 | #4 Play after a win | After X wins, any further `Play` throws "Game over" | Check `Winner()` at the start of `Play` |
| 3.5 | Ordering hazard | Move validation into one guard, so all checks run before `_lastSymbol` or the board changes | Covered by the 3.2 test |

## Phase 4: Optional follow-ups (decide separately)

- **Draw detection:** return or expose a draw when the board is full with no winner.
- **API cleanup:** rename to TypeScript conventions (`play`, `winner`) and return `Player | null` instead of `' '`. This is a breaking change, so all tests must be updated in the same commit.
- **Typed errors:** throw specific error classes (`InvalidMoveError`) instead of bare `Error`, and assert on messages in the tests.
- **Dependencies:** `@types/node` 7 and `@types/jest` 25 are very old. Align them with jest 29.

## Definition of done

- [ ] `npm run build`, `npm test` and `npm run lint` pass with no errors
- [ ] All four review findings are fixed, each with a regression test
- [ ] None of the kata's 14 listed smells remains visibly in `Game.ts`
- [ ] The original 10 tests still pass, unmodified apart from the `var` → `const` change
