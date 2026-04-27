# Scanner — C#

## Required tools

- `roslynator` CLI — complexity + analyzer diagnostics. Verify: `roslynator --version`. Install: `dotnet tool install -g roslynator.dotnet.cli`.
- `coverlet.console` — coverage. Verify: `coverlet --version`. Install: `dotnet tool install -g coverlet.console`.
- `dotnet` SDK must be present; repo must build (`dotnet build -c Debug --nologo -v q`).
- Repo must have a test project (detect via `*.Tests.csproj` / `*Tests/*.csproj`, or any csproj referencing `Microsoft.NET.Test.Sdk`).

If any is missing, refuse and name the missing tool.

## Shortlist heuristic

Goal: ~8 highest-looking candidates without full-repo CRAP computation.

```
roslynator analyze <solution-or-csproj> \
  --severity-level info \
  --analyzer-assemblies Microsoft.CodeAnalysis.NetAnalyzers \
  --output /tmp/yorick-roslynator.xml
```

Filter the report for rule **CA1502** ("Avoid excessive complexity"). Each hit gives `file:line` + method name + complexity. If CA1502 isn't firing enough hits (threshold too high), fall back to parsing method length + nesting depth via a small Roslyn script; combine as `method_LOC × max_nesting_depth`.

Take the top `YORICK_MAX_CANDIDATES` by cyclomatic × method SLOC. Max nesting depth is the tiebreaker.

Exclude `*/obj/*`, `*/bin/*`, generated files (`*.g.cs`, `*.Designer.cs`), and anything under test projects.

## Coverage on the shortlist

Run the test project with coverlet producing per-file/per-line data, scoped to the source files containing shortlisted methods:

```
coverlet <test-assembly>.dll \
  --target "dotnet" \
  --targetargs "test --no-build <test-project>.csproj" \
  --format json \
  --include "[<source-asm>]*" \
  --output /tmp/yorick-cov.json
```

Parse `/tmp/yorick-cov.json` — extract per-file line hits, then compute per-method coverage by intersecting the method's line range with executed lines.

If the test run fails → refuse, citing the failure.

## CRAP computation

```
CRAP = comp² × (1 − cov/100)³ + comp
```

`comp` = cyclomatic from CA1502 / roslynator. `cov` = per-method line-coverage percentage.

## Refactor strategy hints (C#)

- Long method with mixed concerns → extract private methods with clear names.
- Nested `if`/`switch` on type → pattern matching with `switch` expression.
- Parameter-laden method → introduce a parameter object `record`.
- Repeated null-checks → guard clauses / null-coalescing / nullable-aware refactor.
- Async chain with `.Result` / `.Wait()` → fully async/await.

## Test-add strategy hints (C#)

- xUnit `[Theory]` / `[InlineData]` for parametrized characterization — two inline-data rows per commit.
- NUnit equivalent: `[TestCase]`.
- Prefer real inputs over mocks for characterization tests; mock only external boundaries.
