# Reaper Agent Skills & Agent Definitions

**Professional REAPER ReaScript Release Pipeline** — 5 OpenCode agents that take working REAPER Lua scripts from "runs locally" to "published on ReaPack/Gumroad."

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Pipeline Version: 1.0.0](https://img.shields.io/badge/Pipeline-v1.0.0-brightgreen.svg)](https://github.com/scottmills/reaper-agent-skills)
[![REAPER API: Lua 5.3+](https://img.shields.io/badge/REAPER-Lua%205.3+-blue.svg)](https://reaper.fm)
[![Status: active](https://img.shields.io/badge/status-active-green.svg)]

## Overview

This repository contains **5 OpenCode agents** forming a validated release pipeline for REAPER Lua scripts. The pipeline enforces quality gates, proper metadata, and ReaPack indexing so your scripts are ready for public distribution.

> **Pipeline order (binding):** `lua-auditor → header-doctor → rt-optimizer → index-builder`. Never reorder.

## Agents Included

| Agent | Role | Description |
|-------|------|-------------|
| **lua-auditor** | Read-only static audit | Runs luacheck/luac, catches Lua-pattern misuse, regex-isms, nil-guard gaps, undo-block imbalance, and non-existent APIs |
| **header-doctor** | Mechanical reformat | Converts script headers to ReaPack `@metadata` convention, enforces `SGM_` naming |
| **rt-optimizer** | Correctness + perf | Path handling via `GetResourcePath`, Undo/PreventUIRefresh hygiene, defer-loop performance |
| **index-builder** | Index generation | Runs `reapack-index`, generates `index.xml`, declares ReaImGui as dependency (never bundles) |
| **reapack-prep** | Orchestrator | Delegates in order above, pauses for approval before destructive edits and before publishing |

## Quick Start

```bash
# 1. Copy agent files to opencode agents directory
cp reaper-agent-skills/*.md ~/.config/opencode/agents/

# 2. Merge opencode.json into existing config
#    (adds the `agent` block and one `command`)

# 3. Install required tooling (once)
luarocks install luacheck          # or: apt/brew install luarocks first
gem install reapack-index           # Ruby gem for ReaPack index generation

# 4. Run the pipeline on a REAPER Lua script
opencode run --agent reapack-prep "Prep SGM_Script_Name.lua for ReaPack release"
# or: opencode run /reapack-prep SGM_Script_Name.lua
```

## Pipeline Details

### lua-auditor
- **Read-only**: never modifies your script
- Catches: `{n,m}` pattern quantifier bugs, `vim.*` undefined globals, hardcoded `C:\` paths, placeholder/stub bodies
- Output: structured audit report with `NOT-SHIPPABLE` flags for unfixable issues

### header-doctor
- Converts block-comment headers to ReaPack `@metadata` format
- Enforces filename prefix `SGM_`
- Adds `@author Scott Mills`, `@version`, `@changelog`, `@provides`, `@about`

### rt-optimizer
- Replaces hardcoded paths with `reaper.GetResourcePath() + OS branch`
- Ensures Undo_BeginBlock/EndBlock and PreventUIRefresh balance on ALL paths
- Optimizes defer-loop allocation and batch operations

### index-builder
- Validates headers with `reapack-index --check` (gate: never push a failing index)
- Generates/updates `index.xml` via `reapack-index --scan`
- Declares ReaImGui as runtime dependency (never bundled — LGPLv3)
- Two release paths: Free → ReaTeam PR; Paid → own repo + Gumroad

## License

MIT License. See `LICENSE.MIT` for full terms.

Dependencies (luacheck, reapack-index) have their own licenses — see respective documentation.

## Professional Tags

`#reaper` `#lude #audio-plugin` `#reapack` `#gumroad` `#open-source` `#mit-license` `#dsp` `#lua` `#dawhardware`

## Repository Structure

```
reaper-agent-skills/
├── LICENSE.MIT           # MIT license
├── README.md              # This file
├── opencode.json          # Agent definitions + custom command
├── AGENTS.md              # Pipeline overview and hard rules
├── SKILL.md               # REAPER Lua syntax reference
├── reapack-prep.md        # Orchestrator agent
├── header-doctor.md       # Header conversion agent
├── lua-auditor.md         # Read-only static audit agent
├── rt-optimizer.md        # Correctness + perf agent
└── index-builder.md       # ReaPack index generation agent
```

## Pipeline Enforced Rules

- **Never publish a stub** — if a script's core function returns hardcoded/placeholder data, mark NOT-SHIPPABLE and refuse to index
- **Pause before destructive edits and before publishing** — show a diff, get a yes
- **One script at a time** — batch hides regressions
- **Free vs paid** is a release decision; Free → ReaTeam PR, Paid → own repo + Gumroad
- Known bug patterns caught by auditor (see AGENTS.md)

## Badges

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Pipeline: v1.0.0](https://img.shields.io/badge/Pipeline-v1.0.0-brightgreen.svg)
![REAPER: Lua 5.3+](https://img.shields.io/badge/REAPER-Lua%205.3+-blue.svg)
![Status: active](https://img.shields.io/badge/status-active-green.svg)

---

*Built for opencode (markdown-agent format, verified current November 2026). Author: Scott Mills (@author baked into all output.)*
