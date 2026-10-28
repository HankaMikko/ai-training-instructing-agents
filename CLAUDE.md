# CLAUDE.md

Smelly Tic Tac Toe: a code smells refactoring kata. The code lives in `app/`. Review results are in `docs/ReviewFindings.md` and the refactoring roadmap is in `docs/ImprovementPlan.md`.

## Tech stack

- TypeScript 5.9, `strict` mode, compiled to ES6 / CommonJS
- Jest 29 via `ts-jest`; tests live next to the source as `*.test.ts`
- ESLint 8 + `@typescript-eslint` 7 (type-aware). `eqeqeq` and `no-var` are errors
- Node, no runtime libraries

## Commands (run in `app/`)

- `npm test` runs the tests. Run it after every change
- `npm run lint` runs the linter
- `npm run build` compiles with `tsc`

## Architecture rules

1. **`Game` holds the rules and `Board` holds the cells.** Turn order, move checks and win detection go in `Game` (or in rule types split out of it). Tile storage and lookup go in `Board`. Only `Game` is exported; keep everything else internal to the module.
2. **Reject an invalid move by throwing an `Error`, and run every check before changing any state.** A failed `Play()` must leave the game exactly as it was.
3. **Refactor in small steps that keep the tests passing after each one.** Never mix a refactor and a behaviour change in one commit. Fix a bug test-first: write a failing test, then the smallest fix.

## Must not change

1. **The public API:**
   - `new Game()`
   - `Play(symbol: string, x: number, y: number): void`, which throws on an invalid move
   - `Winner(): string`, which returns `'X'`, `'O'` or `' '` (no winner)

   `x` is the row and `y` the column, both 0–2. Change the API only when explicitly asked.
2. **The existing tests in `app/src/Game.test.ts`.** Don't edit, weaken or delete them to make code pass; add new tests instead. The one allowed change is replacing `var` with `const`.
