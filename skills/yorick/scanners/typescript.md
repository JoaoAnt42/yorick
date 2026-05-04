# Scanner — TypeScript

## Required tools

- `ts-complex` (preferred) — cyclomatic complexity for TS/JS. Verify with `ts-complex --version`. Install: `npm i -g ts-complex`.
- Fallback: repo-local `eslint` with the `complexity` rule if `ts-complex` is unavailable.
- Repo must have a test runner configured: vitest (`vitest.config.*` or `"vitest"` in deps) or jest (`jest.config.*` or `"jest"` in deps).

If no complexity tool and no eslint fallback → refuse and name the missing tool.

## Shortlist heuristic

Goal: ~8 highest-looking candidates.

### Path A — `ts-complex` available

Walk the source directory (respect `tsconfig.json` `include`/`exclude`; skip `node_modules`, `dist`, `build`, test files):

```
ts-complex <file> --json
```

per source file, collect all function-level entries, and keep the top `YORICK_MAX_CANDIDATES` by `(cyclomatic × sloc)` as the cheap heuristic. Max nesting depth is a tiebreaker (walk the TS AST via `ts-morph` inline if needed).

### Path B — ESLint fallback (no global complexity tool)

```
npx eslint --no-eslintrc --parser-options=project:./tsconfig.json \
  --rule '{"complexity":["error",1]}' \
  --format json "src/**/*.{ts,tsx}"
```

Collect every violation's reported cyclomatic value. Rank by cyclomatic × function SLOC.

If the repo uses `eslint.config.*` (flat config), use `--config` instead of `--no-eslintrc`.

## Coverage on the shortlist

Run the repo's test runner with coverage, scoped to the files containing shortlisted functions:

- **vitest:** `npx vitest run --coverage --coverage.include="<file1>,<file2>,..." --coverage.reporter=json-summary`
- **jest:** `npx jest --coverage --collectCoverageFrom="<glob>" --coverageReporters=json-summary`

Parse the `coverage-summary.json` (or per-file coverage data) and compute per-function coverage by intersecting the function's line range (from the TS AST) with the file's covered lines.

If the test run fails entirely → refuse, citing the failure.

## CRAP computation

```
CRAP = comp² × (1 − cov/100)³ + comp
```

`comp` = cyclomatic from ts-complex / eslint. `cov` = per-function line-coverage percentage.

## Refactor strategy hints (TypeScript)

- Deeply-nested promise chains → `async/await` + guard clauses.
- Giant `switch` on string union → lookup object `Record<Union, Handler>`.
- Discriminated-union exhaustiveness handled with `if`-chains → exhaustive switch + `never` assertion.
- Over-long React component with multiple `useEffect` → extract custom hooks (one concern per hook).
- Boolean-flag parameter → split into two functions.

## Test-add strategy hints (TypeScript)

- Vitest/jest `.each` for parametrized characterization — two cases per commit.
- For pure functions: snapshot the output if the shape is stable; otherwise explicit `expect().toEqual`.
- Avoid new mocks for characterization — hit the real function, fake only external I/O.
