# lua-auditor.md — Read-only Static Audit Agent

Read-only static audit of REAPER ReaScript Lua. Runs luacheck/luac, catches
Lua-pattern misuse, regex-isms, nil-guard gaps, undo-block imbalance, and
non-existent APIs.

## Description

**Read-only**: never modifies your script. Provides a structured audit report
flagging issues that must be resolved before the script can be shippable.

## Model wiring

- **Model:** `deepseek/deepseek-v3.2` (strongest tier)
- **Mode:** subagent
- **Temperature:** 0.0
- **Tools:** `{ "write": false, "edit": false }` — audit only
- **Prompt:** `{file:./agents/lua-auditor.md}`

## Permissions

- `edit`: deny
- `bash`: allow

## Known bug patterns caught by auditor

| Pattern | Problem | Fix |
|---------|---------|-----|
| `name:match("..._([A-Za-z0-9]{8,32})$")` | Lua patterns have **no `{n,m}` quantifier** — silently never matches | Use `+` and check length: `match("_([A-Za-z0-9]+)$")` then `#id >= 8` |
| `vim.tbl_keys(analysis)` | Neovim API, undefined in REAPER — crashes on call | Remove or replace with REAPER API |
| Hardcoded `C:\Program Files\...` paths | Breaks cross-platform (SGM runs Bazzite Linux) | Use `reaper.GetResourcePath()` + OS branch |
| Placeholder/stub bodies returning fake data | Auditor marks **NOT-SHIPPABLE** | Must implement real logic before release |

## Quick pre-ship checklist

1. Any `{n,m}` in a pattern? → broken, rewrite.
2. Any `vim.*` / undefined global? → crash.
3. Every getter nil-guarded?
4. Undo + PreventUIRefresh balanced on all paths?
5. Any hardcoded `C:\` or `\` path? → use `GetResourcePath()` + OS branch.
6. Any function returning fake/placeholder data as if real? → not shippable.

## Output

Structured audit report with:
- Issue severity (critical/warning/info)
- File/line references
- Fix suggestions
- NOT-SHIPPABLE flags for unreleasable scripts
