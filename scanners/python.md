# Scanner — Python

## Required tools

- `radon` — cyclomatic complexity. Verify with `radon --version`. Install: `pipx install radon`.
- `coverage` — line coverage. Verify with `coverage --version`. Install: `pipx install coverage`.
- Repo must have a pytest configuration (`pyproject.toml` `[tool.pytest.ini_options]`, `pytest.ini`, or `setup.cfg [tool:pytest]`).

If any is missing, refuse and name the missing tool.

## Shortlist heuristic

Goal: ~8 highest-looking candidates without computing CRAP across the whole repo.

```
radon cc -s -a --no-assert --exclude "tests/*,**/test_*.py,**/*_test.py" .
```

`-s` prints the complexity score. Sort the output by score descending and take the top `YORICK_MAX_CANDIDATES`. Each line is `path:line function score (rank)`.

For each candidate, measure function LOC and max nesting depth using the repo's Python AST via a small inline script — this is the tiebreaker when two functions have the same radon score.

## Coverage on the shortlist

Run pytest with coverage scoped to the files containing shortlisted functions:

```
coverage run --source=<comma-separated-shortlist-files> -m pytest -q
coverage json -o /tmp/yorick-cov.json
```

Parse `/tmp/yorick-cov.json` — extract per-file line coverage, then compute per-function coverage by intersecting the function's line range (from AST) with the file's executed lines.

If pytest fails entirely: refuse, citing the test failure.

## CRAP computation

For each shortlisted function:

```
CRAP = comp² × (1 − cov/100)³ + comp
```

Where `comp` is the radon score and `cov` is the per-function line-coverage percentage.

## Refactor strategy hints (Python)

- Deeply nested `if`/`for`/`try` → extract helpers, guard-clause early returns.
- Long `match`/`if-elif` chain → dispatch dict.
- Mixed concerns (I/O + logic) → split pure from impure.
- Boolean-flag parameters → split into two functions.

## Test-add strategy hints (Python)

- Use `pytest.mark.parametrize` to cover branches cheaply.
- Characterization tests: run the function, capture the output, assert it. Two parametrize cases per commit.
