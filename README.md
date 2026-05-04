# Yorick

A Claude Code skill that finds your highest-CRAP function, reduces it, and opens a small focused PR.

Supports **C#**, **Python**, **TypeScript**, and **JavaScript** (plain JS, ESM, CJS, and Google Apps Script).

## Install

```
/plugin marketplace add JoaoAnt42/yorick
/plugin install yorick@yorick
```

That's it — the `yorick` skill and `/yorick` command are now available.

### Manual install (no plugin system)

```bash
git clone https://github.com/JoaoAnt42/yorick
cd yorick
cp -r skills/yorick ~/.claude/skills/yorick
cp commands/yorick.md ~/.claude/commands/yorick.md
```

## Use

From a git repo root:

```
/yorick .
```

Yorick will detect the stack, shortlist high-CRAP candidates (or full-scan on small repos), pick the best target, and open a PR — or exit cleanly if nothing meets the threshold.

## Configuration

| Env var | Default | Effect |
|---|---|---|
| `YORICK_CRAP_THRESHOLD` | `30` | Minimum CRAP score to trigger a PR |
| `YORICK_MAX_CANDIDATES` | `8` | Functions to shortlist |
| `YORICK_FULL_SCAN` | auto | `true` forces full-scan; `false` forces heuristic. Auto-triggers on ≤30 source files. |
| `YORICK_INTRODUCE` | `true` | Post a neutral intro comment on the first PR |

## Requirements

- Claude Code with plugin / skill support
- `gh` CLI authenticated
- Stack tooling: `pytest`/`coverage` (Python), `vitest`/`jest`/`mocha` (TS/JS), `dotnet test` (C#). Google Apps Script uses manual `Tests_*.gs` harness; CI requirement waived in GAS mode.
