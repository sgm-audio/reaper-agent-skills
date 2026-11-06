# AGENTS.md — REAPER ReaScript Release Pipeline

This repository contains **5 OpenCode agents** forming a pipeline that takes working REAPER Lua scripts from "runs locally" to "published on ReaPack/Gumroad."

## Pipeline order (must run in this sequence)

```
reapack-prep (orchestrator)
  ├─▶ lua-auditor      (read-only) — static analysis, finds real bugs first
  ├─▶ header-doctor    — converts headers to ReaPack @metadata, enforces SGM_ naming
  ├─▶ rt-optimizer     — path handling, undo hygiene, defer-loop performance
  └─▶ index-builder    — runs reapack-index, generates index.xml, declares deps
```

**Run order is binding:** audit → header → optimize → index. Do not reorder.

## Install agents

```bash
# Copy agent files to your opencode agents dir
cp *.md ~/.config/opencode/agents/

# Merge opencode.json into ~/.config/opencode/opencode.json
# (only adds the `agent` block and one `command`)
```

## Required external tooling (install once)

```bash
# luacheck + luac (Lua 5.3+)
luarocks install luacheck    # or: apt/brew install luarocks first

# reapack-index (Ruby gem)
gem install reapack-index
```

## Run the pipeline

```bash
# Whole pipeline on one script
opencode run --agent reapack-prep "Prep SGM_Suno_ZIP_Importer.lua for ReaPack release"

# Or the custom command (defined in opencode.json)
opencode run /reapack-prep SGM_Suno_ZIP_Importer.lua
```

The orchestrator delegates in order, surfaces each agent's findings, and **pauses for your approval before any destructive edit and before publishing the index.**

## Agent model wiring (placeholders — swap for your configured providers)

| Agent          | Role                  | Suggested tier       | Example string           |
| -------------- | --------------------- | -------------------- | ------------------------ |
| reapack-prep   | orchestrate, decide   | strong               | `deepseek/deepseek-v3.2` |
| header-doctor  | mechanical reformat   | fast/cheap           | `qwen/qwen3.5-coder`     |
| lua-auditor    | find real bugs        | **strongest you have** | `deepseek/deepseek-v3.2` |
| rt-optimizer   | correctness + perf    | strong               | `glm/glm-4.6`            |
| index-builder  | follow spec exactly   | mid                  | `kimi/kimi-k2`           |

> The auditor is the one place to spend your best model. A weak model misses the `{8,32}` class of bug.

## Hard rules the orchestrator enforces

- **Never publish a stub.** If a script's core function returns hardcoded/placeholder data (markers: "placeholder", "simulate", "In real implementation", fake confidence like `0.95`), mark **NOT-SHIPPABLE** and refuse to index.
- **Pause before destructive edits and before publishing.** Show a diff, get a yes.
- **One script at a time** unless told otherwise. Batch hides regressions.
- **Free vs paid is a release decision, not the agent's.** Surface which tier the script targets; if unstated, ask. Free → ReaTeam PR. Paid → SGM's own repo + Gumroad.

## Known bug patterns in SGM's current codebase (found by hand)

The auditor will catch these, but knowing them upfront saves cycles:

| Pattern | Problem | Fix |
|---------|---------|-----|
| `name:match("..._([A-Za-z0-9]{8,32})$")` | Lua patterns have **no `{n,m}` quantifier** — silently never matches | Use `+` and check length: `match("_([A-Za-z0-9]+)$")` then `#id >= 8` |
| `vim.tbl_keys(analysis)` | Neovim API, undefined in REAPER — crashes on call | Remove or replace with REAPER API |
| Hardcoded `C:\Program Files\...` paths | Breaks cross-platform (SGM runs Bazzite Linux) | Use `reaper.GetResourcePath()` + OS branch |
| Placeholder/stub bodies returning fake data | Auditor marks **NOT-SHIPPABLE** | Must implement real logic before release |

## ReaPack metadata conventions (header-doctor enforces)

```lua
-- @description Suno ZIP Importer
-- @author Scott Mills          -- ALWAYS this exact string
-- @version 1.1.0               -- semver, carry forward from old header if present
-- @changelog
--   Initial ReaPack release
-- @provides
--   [main] .                    -- single file; list companion files if any
-- @about
--   # Suno ZIP Importer
--   Drop a Suno AI export ZIP... Requires ReaImGui (install via ReaPack).
```

- Filename convention: `Various/SGM_Script Name.lua` (prefix `SGM_`)
- Dependencies (ReaImGui, SWS, js_ReaScriptAPI) go in `@about` as human note + runtime guard in code — **never bundled** (LGPLv3)

## Index generation (index-builder)

```bash
cd <repo>
gem list -i reapack-index || gem install reapack-index
reapack-index --check          # validate headers WITHOUT writing (gate)
reapack-index --scan           # generate/update index.xml
git add -A && git commit -m "Index SGM ReaScripts"
```

- `--check` is the gate: never push an index that fails it.
- Two repos: Free → ReaTeam PR (fork `ReaTeam/ReaScripts`). Paid → SGM's own repo, `index.xml` served raw on GitHub, URL delivered to Gumroad buyers.

## Repo structure

```
├── opencode.json           # Agent definitions + custom command
├── SKILL.md                # reaper-lua-syntax skill (reference for agents)
├── reapack-prep.md         # Orchestrator agent
├── header-doctor.md        # Header conversion agent
├── lua-auditor.md          # Read-only static audit agent
├── rt-optimizer.md         # Correctness + perf agent
└── index-builder.md        # ReaPack index generation agent
```

## Lua version & REAPER API notes

- REAPER embeds **Lua 5.3+** (bitwise ops `& | ~ << >>`, `//` integer division work)
- Global API: `reaper.*`, `gfx.*` — no `vim`, no `os.*` surprises
- **Lua tables are 1-indexed**; **REAPER object collections are 0-indexed** (tracks, items, FX, sends). Mixing = off-by-one bugs.
- **Nil is normal** — guard every getter (`GetSelectedMediaItem`, `GetActiveTake`, `GetTrack`, `GetMediaItemTake_Source` all return nil legitimately).
- **Undo + PreventUIRefresh must balance on ALL paths** (early returns, errors). Unbalanced = frozen UI or corrupted undo history.
- **Lua patterns ≠ regex** — no `{n,m}`, no `\d/\w/\s`, no alternation `(a|b)`. Only `* + - ?` quantifiers.

## Quick pre-ship checklist (from SKILL.md)

1. Any `{n,m}` in a pattern? → broken, rewrite.
2. Any `vim.*` / undefined global? → crash.
3. Every getter nil-guarded?
4. Undo + PreventUIRefresh balanced on all paths?
5. Any hardcoded `C:\` or `\` path? → use `GetResourcePath()` + OS branch.
6. Any function returning fake/placeholder data as if real? → not shippable.
