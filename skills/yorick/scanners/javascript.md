# Scanner — JavaScript (vanilla JS, Node, Google Apps Script)

Covers: `.js`, `.mjs`, `.cjs`, `.gs` (Google Apps Script).

GAS note: `.gs` files are JavaScript at the language level — same parsers/complexity tools work. The constraint is the runtime: no npm test runner, no coverage. The scanner adapts (see "GAS mode" below).

## Required tools

Resolution order (use the first that works):
1. Globally installed `cr` (`complexity-report`) — verify with `cr --version`.
2. Globally installed `escomplex`.
3. On-demand `npx --yes complexity-report` — works without global install if the network is reachable.
4. Repo-local `eslint` with the `complexity` rule (in standard JS repos that already pin eslint).

If all four fail → refuse and name what was tried.

For non-GAS repos: a test runner must also be configured (vitest, jest, or mocha — detect via `vitest.config.*`, `jest.config.*`, `.mocharc.*`, or deps in `package.json`).

## GAS mode (auto-detected)

A repo is GAS-mode when **any** of:
- `appsscript.json` exists at repo root.
- `.gs` files outnumber `.js`/`.mjs`/`.cjs` files.

GAS mode differences:
- No `package.json` / `node_modules` requirement.
- No CI requirement (preflight relaxes the CI check for GAS).
- "Test suite" = presence of any file matching `Tests_*.gs`, `*_test.gs`, or `*Test*.gs`. If none exist → refuse.
- Coverage is **not measured** — set `cov = 0` for all candidates. CRAP collapses to `comp² + comp`.
- Strategy is **refactor-only**. Do not pick "add tests" — there is no harness to run them in CI. (Yorick may still extend a `Tests_*.gs` file with a characterization test that the user runs manually, but this is a refactor with a safety-net test, not a test-add PR.)
- Run characterization tests by porting the `.gs` function to a temporary Node file and exercising it with vitest/jest in the worktree, OR by invoking via `clasp run` if the repo has `.clasp.json` configured. If neither path works, run the characterization test inline with a hand-rolled assert harness (`node --check` + a small runner) **before** the refactor commit, and again after — discard the worktree if either run fails.

## Source-file exclusions

In addition to the SKILL.md global excludes (`node_modules`, `dist`, `build`, etc.), JS scanning also excludes:

- `Tests_*.gs`, `Tests_*.js`, `Tests_*.mjs` (Apps Script / convention-based test files).
- `*.test.js`, `*.spec.js`, `*.test.mjs`, `*.spec.mjs`, `*.test.ts`-derived counterparts.
- `__tests__/**`, `tests/**`, `test/**`.

## Full-scan mode (small repo)

Per SKILL.md, when ≤30 in-scope source files exist, skip the heuristic shortlist and treat every function in scope as a candidate. Use the same complexity tool, just on every file. Continue with normal CRAP computation.

## Shortlist heuristic (large repos)

Goal: ~8 highest-looking candidates.

### Path A — `complexity-report` / `escomplex` available

Walk source files (skip `node_modules`, `dist`, `build`, `Tests_*.gs`, `*_test.*`, `tests/`):

```
cr --format json <file>
```

Collect all function-level entries, keep top `YORICK_MAX_CANDIDATES` by `(cyclomatic × sloc)`.

### Path B — ESLint fallback

Standard JS:

```
npx eslint --no-eslintrc \
  --rule '{"complexity":["error",1]}' \
  --format json "src/**/*.{js,mjs,cjs}"
```

GAS (no `src/`, files at repo root, no eslint config typically):

```
npx --yes eslint --no-eslintrc \
  --parser-options=ecmaVersion:2020,sourceType:script \
  --rule '{"complexity":["error",1]}' \
  --format json "*.gs"
```

`.gs` files are not recognized by eslint by default — symlink or copy to `.js` in a temp directory inside the worktree before running, then map results back to original paths.

Collect every violation's reported cyclomatic value. Rank by `cyclomatic × function SLOC`.

## Coverage on the shortlist

### Standard JS

Run the repo's test runner with coverage, scoped to files containing shortlisted functions:

- **vitest:** `npx vitest run --coverage --coverage.include="<file1>,<file2>,..." --coverage.reporter=json-summary`
- **jest:** `npx jest --coverage --collectCoverageFrom="<glob>" --coverageReporters=json-summary`
- **mocha + c8:** `npx c8 --reporter=json-summary --include="<glob>" mocha`

Parse `coverage-summary.json`. Compute per-function coverage by intersecting the function's line range (parse with acorn or @babel/parser) with the file's covered lines.

If the test run fails entirely → refuse, citing the failure.

### GAS mode

Skip. `cov = 0` for every candidate.

## CRAP computation

```
CRAP = comp² × (1 − cov/100)³ + comp
```

Standard JS: `cov` = per-function line-coverage percentage.
GAS: `cov = 0` always, so `CRAP = comp² + comp`.

## Refactor strategy hints (JavaScript)

- Deeply-nested callback / promise chains → `async/await` + guard clauses.
- Giant `switch` / `if`-chain on string discriminator → lookup object `{ key: handler }`.
- Long parameter lists → options object.
- Boolean-flag parameter → split into two functions.
- Repeated `try/catch` boilerplate around fetch/HTTP → extract a helper.
- GAS-specific: long `getConfig` / `getSchema` / `getData` functions in Looker Studio connectors → extract per-section builders; collapse repeated `addField().setId().setName().setType()` into a config-driven loop.
- GAS-specific: `PropertiesService` / `CacheService` access scattered throughout a function → extract a single accessor.

## Test-add strategy hints (JavaScript)

Standard JS only (skip in GAS mode):

- Vitest/jest `.each` for parametrized characterization — two cases per commit.
- For pure functions: explicit `expect().toEqual` over snapshots unless the shape is genuinely stable.
- Avoid new mocks for characterization — hit the real function, fake only external I/O (network, filesystem, `Date.now`).
