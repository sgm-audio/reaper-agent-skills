# index-builder.md — ReaPack Index Generation Agent

Builds/validates a ReaPack repo: reapack-index, index.xml, @provides, declares
ReaImGui as a dependency (never bundles it).

## Description

Final release step in the pipeline. Runs `reapack-index` to validate headers
and generate the `index.xml` that makes scripts discoverable on ReaPack.

## Model wiring

- **Model:** `kimi/kimi-k2` (mid tier)
- **Mode:** subagent
- **Temperature:** 0.0
- **Prompt:** `{file:./agents/index-builder.md}`

## Permissions

- `edit`: allow
- `bash`: allow

## Index generation workflow

```bash
cd <repo>
gem list -i reapack-index || gem install reapack-index
reapack-index --check          # validate headers WITHOUT writing (gate)
reapack-index --scan           # generate/update index.xml
git add -A && git commit -m "Index SGM ReaScripts"
```

## Gates and requirements

- **`--check` is the gate**: never push an index that fails validation.
  This catches broken `@metadata`, missing `@provides`, bad version strings,
  and other header issues before they reach users.

- **Two release paths**:
  - **Free** → ReaTeam PR (fork `ReaTeam/ReaScripts`, submit PR for review)
  - **Paid** → SGM's own repo, `index.xml` served raw on GitHub, URL delivered
    to Gumroad buyers after purchase

- **Dependencies**: ReaImGui declared as runtime dependency in `@about` section
  — **never bundled** (LGPLv3). Users must have ReaImGui installed via ReaPack.

- **@provides convention**:
  - `[main] .` — single main file; list companion files if any
  - Companion files listed individually if present

## Repo state after index build

- `index.xml` generated and committed
- All script headers validated
- Dependencies documented
- Release path decided (Free → PR, Paid → own repo)
