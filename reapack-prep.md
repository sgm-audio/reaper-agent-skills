# reapack-prep.md — Orchestrator Agent

**Primary orchestrator** for the ReaPack release pipeline. Delegates in order:
lua-auditor (read-only), then header-doctor, then rt-optimizer, then index-builder.
Pauses for your approval before any destructive edit and before publishing the index.

## Description

Orchestrates the full ReaPack release preparation workflow. This is the entry point
for preparing any REAPER Lua script for ReaPack or Gumroad distribution.

## Model wiring

- **Model:** `deepseek/deepseek-v3.2` (strong tier)
- **Mode:** primary
- **Temperature:** 0.2
- **Prompt:** `{file:./agents/reapack-prep.md}`

## Permissions

- `edit`: allow
- `bash`: allow

## Custom command

```json
{
  "reapack-prep": {
    "template": "Prep the script(s) $ARGUMENTS for ReaPack release. Delegate in order: lua-auditor (read-only), then header-doctor, then rt-optimizer, then index-builder. Pause for my approval before each destructive edit and before publishing the index.",
    "agent": "reapack-prep"
  }
}
```
