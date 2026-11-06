---
name: reaper-lua-syntax
description: REAPER ReaScript Lua syntax, idioms, and the gotchas that cause silent failures. Use this whenever writing, reviewing, debugging, or generating Lua for REAPER — ReaScripts, defer/ImGui loops, track/item/FX manipulation, peak analysis, or anything calling the reaper.* API. Especially use it when a script "runs but does nothing," when patterns/matches misbehave, or before shipping any .lua to ReaPack.
---

# REAPER ReaScript Lua

REAPER embeds Lua 5.3+ (bitwise `& | ~ << >>` work; `//` integer division works).
The whole API lives under the injected global `reaper.*`; the GFX API under
`gfx.*`. There is no `vim`, no `os.*` sandbox surprises beyond the usual, and
no package ecosystem unless an extension (SWS, js_ReaScriptAPI, ReaImGui) is
installed.

## The thing that breaks the most scripts: Lua patterns are NOT regex

This single misconception accounts for most "the script runs but produces
nothing" bugs.

- **No `{n,m}` quantifiers.** `[A-Za-z0-9]{8,32}` does not mean "8 to 32 chars."
  Lua reads `{`, `8`, `,`, `3`, `2`, `}` as literal members/characters. The
  match returns `nil`. → Use `+` and check length in code:
  ```lua
  local id = name:match("_([A-Za-z0-9]+)$")
  if id and #id >= 8 and #id <= 32 then ... end
  ```
- **Character classes use `%`, not `\`:** `%d %a %w %s %l %u %p` (and uppercase
  = complement: `%D` = non-digit).
- **No alternation** `(a|b)`. Match each alternative separately, or use a
  table of patterns.
- **Quantifiers:** only `*` (greedy 0+), `+` (greedy 1+), `-` (lazy 0+),
  `?` (optional). That's it.
- **Magic characters** to escape with `%`: `( ) . % + - * ? [ ] ^ $`.
- **Anchors:** `^` start, `$` end. Captures with `( )`. `.` = any char.

When you genuinely need regex semantics, do it in two passes (match loosely,
then validate/branch in Lua) — don't fake it with pattern syntax that doesn't exist.

## Indexing: the 0 vs 1 split

- **Lua tables are 1-indexed.** `t[1]` is first; `#t` is length; `ipairs`
  iterates 1..#t.
- **REAPER object collections are 0-indexed.** Tracks, items, FX, sends,
  markers: `for i = 0, reaper.CountTracks(0) - 1 do local tr = reaper.GetTrack(0, i)`.
- Mixing these is the classic off-by-one. When you copy REAPER objects into a
  Lua table, you cross from 0-based to 1-based — be deliberate at the boundary.

## Nil is normal — guard every getter

These return `nil` legitimately and dereferencing nil throws mid-execution
(which, inside an undo block, leaves the project in a half-edited state):

```lua
local item = reaper.GetSelectedMediaItem(0, 0); if not item then return end
local take = reaper.GetActiveTake(item);         if not take then return end
local src  = reaper.GetMediaItemTake_Source(take); if not src then return end
```

Also guard extension APIs before first call:
```lua
if not reaper.ImGui_CreateContext then
  reaper.MB("This script needs ReaImGui (install via ReaPack).", "Missing dependency", 0)
  return
end
```

## State hygiene (do this on every mutating script)

```lua
local cur = reaper.GetCursorPosition()
local s, e = reaper.GetSet_LoopTimeRange(false, false, 0, 0, false)

reaper.Undo_BeginBlock()
reaper.PreventUIRefresh(1)

-- ... edits ...

reaper.PreventUIRefresh(-1)
reaper.Undo_EndBlock("My Script v1.0.0", -1)
reaper.SetEditCurPos(cur, false, false)
reaper.GetSet_LoopTimeRange(true, false, s, e, false)
reaper.UpdateArrange()
```

**Both Begin/End and PreventUIRefresh ±1 must balance on every exit path** —
early `return`, `goto`, error. Unbalanced PreventUIRefresh = frozen UI;
unbalanced undo = corrupted undo history. If a function has early returns,
either restructure to a single exit or wrap the body in `pcall` and clean up
after.

## Method calls: `:` vs .

`obj:method(x)` passes `obj` as the hidden first arg `self`; `obj.method(x)`
does not. OOP-style ReaScript (metatables + `__index`) relies on `:` for calls
and `:` in definitions. A `.`/`:` mismatch silently passes the wrong `self`.

## defer / GUI loops

`reaper.defer(fn)` re-runs `fn` next cycle (~30–60 Hz) — that's how ImGui and
animated scripts stay alive. Inside a defer loop:
- **Do not allocate per frame.** Building tables or concatenating strings every
  frame causes GC stutter. Hoist constants outside the loop; keep mutable state
  in one table created once.
- Exit by simply not calling `defer` again (e.g., when the ImGui window closes),
  then `reaper.ImGui_DestroyContext(ctx)`.

## Common API string/number traps

- `reaper.GetSetMediaTrackInfo_String(tr, "P_NAME", name, true)` — the `true`
  means *set*; `false` means *get*. Easy to flip.
- `string.format` for any numeric display; `%%` for a literal percent.
- `tonumber(x)` returns nil on bad input — guard before arithmetic.
- Colors: REAPER wants native ints. `reaper.ColorToNative(r,g,b) | 0x1000000`
  (the high bit flags "color is set").

## Quick self-check before shipping

1. Any `{n,m}` in a pattern? → broken, rewrite.
2. Any `vim.*` / undefined global? → crash.
3. Every getter nil-guarded?
4. Undo + PreventUIRefresh balanced on all paths?
5. Any hardcoded `C:\` or `\` path? → use `GetResourcePath()` + OS branch.
6. Any function returning fake/placeholder data as if real? → not shippable.
