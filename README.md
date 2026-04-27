# Yorick

A Claude Code skill that finds your highest-CRAP function, reduces it, and opens a small focused PR.

Supports **TypeScript**, **Python**, and **C#**.

## Install

```bash
cp -r . ~/.claude/skills/yorick
```

## Use

From a git repo root:

```
/yorick .
```

Yorick will detect the stack, shortlist high-CRAP candidates, pick the best target, and open a PR — or exit cleanly if nothing meets the threshold.

## Configuration

| Env var | Default | Effect |
|---|---|---|
| `YORICK_CRAP_THRESHOLD` | `30` | Minimum CRAP score to trigger a PR |
| `YORICK_MAX_CANDIDATES` | `8` | Functions to shortlist |
| `YORICK_INTRODUCE` | `true` | Post a neutral intro comment on the first PR |

## Requirements

- Claude Code with skill support
- `gh` CLI authenticated
- Stack tooling: `pytest`/`coverage` (Python), `vitest`/`jest` (TS), `dotnet test` (C#)
