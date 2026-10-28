# Review Findings: `app/`

**Date:** 2026-10-28
**Skill:** `code-review:code-review` (Anthropic plugin)
**Scope:** whole `app/` directory (`src/Game.ts`, `src/Game.test.ts`, project config). There was no pull request, so the review covered the code as it stands.

## How the review ran

1. **CLAUDE.md compliance:** skipped. The repository has no `CLAUDE.md` yet.
2. **Bug review:** two independent Opus agents. One looked for obvious bugs and the other for logic and input-handling problems.
3. **Validation:** each reported issue was traced against the source and kept only if a concrete failing move sequence exists.
4. **Filtering:** high-signal issues only. Style issues, code smells and anything a linter catches were excluded, as the skill specifies.

## Validated issues

### 1. `Winner()` checks rows only, never columns or diagonals

- **Where:** [app/src/Game.ts:27-67](../app/src/Game.ts#L27-L67)
- **Type:** bug
- **Problem:** `Winner()` checks only the three rows (x = 0, 1, 2). Wins down a column or along either diagonal are never detected.
- **Failure scenarios:**
  - Column: X(0,0), O(0,1), X(1,0), O(1,1), X(2,0). Expected `'X'`; actual `' '`.
  - Diagonal: X(0,0), O(0,1), X(1,1), O(0,2), X(2,2). Expected `'X'`; actual `' '`.
  - Anti-diagonal: X(0,2), O(0,0), X(1,1), O(0,1), X(2,0). Expected `'X'`; actual `' '`.
- **Why the tests miss it:** `Game.test.ts` only has row-win tests.

### 2. Out-of-range coordinates crash and leave the game in a broken state

- **Where:** [app/src/Game.ts:18](../app/src/Game.ts#L18), [app/src/Game.ts:23-24](../app/src/Game.ts#L23-L24), [app/src/Game.ts:93-102](../app/src/Game.ts#L93-L102)
- **Type:** bug
- **Problem:** nothing checks the bounds. `TileAt` and `AddTileAt` use `find(...)!`, which returns `undefined` when no tile matches.
- **Failure scenarios:**
  - First move `Play('X', 3, 3)`: line 23 sets `_lastSymbol = 'X'`, then line 101 throws `TypeError`. Nothing is placed, but the game now thinks X has played. Retrying `Play('X', 0, 0)` throws "Invalid next player", and O can move before X has placed a piece.
  - Later move `Play('O', -1, 5)`: line 18 throws a raw `TypeError` instead of "Invalid position".

### 3. Any symbol is accepted as a player

- **Where:** [app/src/Game.ts:7-16](../app/src/Game.ts#L7-L16)
- **Type:** bug (invalid input leads to wrong state)
- **Problem:** the first move only rejects `'O'`. Later moves only reject a repeat of the last symbol. Nothing restricts players to `'X'` and `'O'`.
- **Failure scenarios:**
  - `Play('Z', 0, 0)` is accepted as the first move.
  - After X moves, `Play('Q', 1, 1)` is accepted.
  - `Play(' ', 0, 0)` writes `' '` to the board and leaves `_lastSymbol` as `' '`. The game still treats the next move as the first one, and the tile still looks empty.

### 4. Moves are accepted after the game has been won

- **Where:** [app/src/Game.ts:5-25](../app/src/Game.ts#L5-L25)
- **Type:** logic error
- **Problem:** `Play()` never checks whether there is already a winner. Because `Winner()` checks row 0 first, a later win can replace an earlier one.
- **Failure scenario:** X(1,0), O(0,0), X(1,1), O(2,2), X(1,2). X wins the middle row and `Winner()` returns `'X'`. Then O(0,1), X(2,0), O(0,2) are all accepted, and `Winner()` returns `'O'`.

## Excluded

- **`npm run lint` fails:** `.eslintrc.json` sets `eqeqeq: error`, and every comparison in `Game.ts` uses `==` or `!=`. The skill excludes issues a linter catches. These failures will show up as soon as lint runs.
- **`outDir: ./dist` vs `main: ./build/index.js`:** the package entry point is wrong, but this doesn't affect build or tests.
- **Code smells:** for example duplicated row checks, primitive obsession (`string` symbols, `' '` meaning empty), comments that restate the code, the unused `tile` variable in `AddTileAt`, and the data-class `Tile`. These are what the kata is about, and the skill leaves quality concerns out of a bug review. They belong in `docs/ImprovementPlan.md`.

## Tests

All 10 existing test cases in `Game.test.ts` use legal move sequences and assert correct results. No test asserts wrong behaviour. None of them covers columns, diagonals, invalid coordinates, invalid symbols or play after a win. The tests were checked by reading them, not run, because `node_modules` isn't installed.
