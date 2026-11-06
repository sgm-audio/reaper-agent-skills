# header-doctor.md — Header Conversion Agent

Converts script headers to the ReaPack `@metadata` convention and enforces
`author_Name.lua` naming.

## Description

Mechanical reformatting agent that transforms block-comment headers in REAPER
Lua scripts into the ReaPack `@metadata` format. Enforces consistent naming,
author attribution, and metadata fields.

## Model wiring

- **Model:** `qwen/qwen3.5-coder` (fast/cheap tier)
- **Mode:** subagent
- **Temperature:** 0.0
- **Prompt:** `{file:./agents/header-doctor.md}`

## Permissions

- `edit`: allow
- `bash`: deny

## ReaPack metadata conventions enforced

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
- Dependencies (ReaImGui, SWS, js_ReaScriptAPI) go in `@about` as human note
  + runtime guard in code — **never bundled** (LGPLv3)
