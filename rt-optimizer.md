# rt-optimizer.md — Correctness + Performance Agent

Correctness and performance pass for REAPER Lua: GetResourcePath path handling,
Undo/PreventUIRefresh hygiene, batch operations, defer-loop allocation.

## Description

Correctness + performance agent that ensures path handling uses REAPER's
native APIs, undo/PreventUIRefresh balances on all execution paths, and
defer-loop performance is optimized for real-time audio safety.

## Model wiring

- **Model:** `glm/glm-4.6` (strong tier)
- **Mode:** subagent
- **Temperature:** 0.1
- **Prompt:** `{file:./agents/rt-optimizer.md}`

## Permissions

- `edit`: allow
- `bash`: deny

## Key areas optimized

### Path handling

- Replaces hardcoded `C:\Program Files\...` paths with `reaper.GetResourcePath()`
- OS-specific branches for Windows vs cross-platform compatibility
- Uses `reaper.GetResourcePath()` for all resource location

### Undo / PreventUIRefresh hygiene

- Ensures `Undo_BeginBlock()` / `Undo_EndBlock()` balance on **every** exit path
  (early returns, `goto`, errors, exceptions)
- Ensures `PreventUIRefresh(1)` / `PreventUIRefresh(-1)` balance on **every**
  exit path — unbalanced = frozen UI or corrupted undo history
- Restructures functions with early returns to single-exit pattern or wraps in
  `pcall` with cleanup after

### defer-loop performance

- Hoists constant table allocations outside defer loops
- Prevents per-frame string concatenation that causes GC stutter
- Keeps mutable state in one table created once, outside the loop
- Exit pattern: simply stop calling `defer` when ImGui window closes, then
  `reaper.ImGui_DestroyContext(ctx)`

### Batch operations

- Groups multiple API calls where possible
- Minimizes `reaper.UpdateArrange()` calls
- Uses `reaper bulb` operations only when necessary
