# Profiler: the companion agent plugin

Status: design, 2026-11-14. Not built.

## Why an agent, and only here

One fixed template for "what to back up" fails, because machines differ too much (see [recipe-ubuntu.md](recipe-ubuntu.md)). Separating intent from noise (673 manual apt packages, 170 enabled units) takes judgement and a conversation with the owner. That is agent work.

Everything after that is not. So the split is:

| Step | Who | Deterministic? |
|---|---|---|
| First profile of a machine: classify what's there, propose the machine's footprint config | **Profiler plugin** (Claude Code) with the owner | No: judgement, asks questions |
| Capture, push, drift, status, restore, test | **`footprint` CLI** | Yes: no AI needed, works offline from the agent |
| Re-profile when `drift` shows many unclassified items | Profiler plugin | No |

Restore must never depend on an AI service being available.

## What the profiler does

1. Runs the read-only collectors for the machine's formula (e.g. Ubuntu layers 0–14) via `footprint inspect --json`.
2. Classifies every item into one of:
   - **declare**: intent, goes in the playbook (installed on purpose, still wanted)
   - **already-synced**: lives in a cloud service; record only how to reconnect (e.g. "sign in to the sync client")
   - **sidecar**: local-only data that must travel with the footprint
   - **secret**: goes in the encrypted blob
   - **ignore**: distro default, dependency, cache, hardware-specific, or an abandoned experiment
3. Uses evidence for each call: apt seed manifest, install dates (`/var/log/apt/history.log`, `/var/log/dpkg.log`), last-used times, whether a package is pulled in by another, whether a repo has a remote.
4. Asks the owner only about the ambiguous ones, in batches ("these 40 packages were installed once in 2025 and never updated: keep, drop, or decide one by one?").
5. Writes the result into the footprint repo: the machine's block in `footprint.yml`, per-layer include/ignore lists, and a `PROFILE.md` recording why each non-obvious call was made.
6. Runs a first `footprint capture` and shows the resulting playbook size per layer.

## Packaging

- The plugin ships in this repo (`profiler-plugin/`), so the CLI and its companion version together.
- It calls the CLI and never writes playbooks directly. The CLI owns the format.
- The plugin itself is optional. A user can write the include/ignore lists by hand and get the same deterministic CLI.
