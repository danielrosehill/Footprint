# Footprint design (draft, 2026-11-14)

Status: design notes only. Nothing here is implemented. "Decided" means agreed in the founding discussion; "proposed" means a default open to change.

## 1. Principle (decided)

The user's computer is ephemeral. Footprint is `requirements.txt` for the filesystem: a declarative record of what the machine is, plus a small sidecar of local-only data. It is a **disaster-recovery** tool for a one-time full restore onto a fresh OS install. It is not for rollback and keeps no restore points: there is one rolling copy.

## 2. Registry: `footprint.yml` (proposed)

```yaml
version: 1
targets:
  wasabi:   { remote: "wasabi-crypt:footprint", kind: cloud, offsite: true }
  b2:       { remote: "b2-crypt:footprint",     kind: cloud, offsite: true }
  nas:      { remote: "nas-sftp:/volume1/footprint", kind: nas, offsite: false }
policy:
  min_targets: 2          # 3-2-1: live machine + >=2 targets
  require_offsite: 1      # at least one target must be off-site
machines:
  laptop:
    base: laptop          # machine-base prefix on each target
    targets: [wasabi, nas]
    distro: ubuntu-24.04  # base the footprint expects to be replayed onto
    sidecar:
      include: ["~/Documents/local-only", "~/.local/share/some-app"]
      exclude: ["**/node_modules", "**/.venv", "**/__pycache__"]
  desktop:
    base: desktop
    targets: [wasabi, b2]
```

`remote` values are rclone remotes, so target credentials live in rclone's own config (or env), never in the footprint repo.

## 3. What gets captured (proposed)

Each source is a **collector** that outputs a sorted, normalised list. That list is the desired state.

| Collector | Source on live system | Ansible on restore |
|---|---|---|
| user | username, shell, groups | `ansible.builtin.user` |
| apt-manual | `apt-mark showmanual` minus the distro's default manifest | `apt` |
| apt-sources | `/etc/apt/sources.list.d/*`, keyrings | `deb822_repository` / copy |
| flatpak | `flatpak list --app --columns=application,origin` + remotes | `community.general.flatpak` |
| snap | `snap list` (non-base) | `community.general.snap` |
| uv-tools / pipx | `uv tool list`, `pipx list --json` | `command` with `creates:` |
| conda | `conda env list` + `conda env export --from-history` | `conda env create -f` |
| npm-global, cargo | `npm ls -g --json`, `~/.cargo/.crates2.json` | `npm` / `command` |
| repos | every git checkout under configured roots: path + remote URL + branch | `ansible.builtin.git` (clone only, never content) |
| services | enabled systemd units, system and `--user`, that differ from distro defaults | `systemd_service` |
| dotfiles | an explicit include list, copied into the footprint repo | `copy` / `template` |
| etc-deltas | conffiles that differ from the package version (`dpkg --verify`) | `copy` |
| desktop settings | e.g. `dconf dump /` or KDE rc files on an include list | `dconf load` / `copy` |

Git repos are recorded by remote, not backed up. They are already on GitHub (the principle). A checkout with unpushed commits or no remote is flagged in `drift`, not silently copied.

## 4. Incremental capture and removals (decided: the machine is the source of truth)

`footprint capture` compares each collector's live output with the stored list, shows a diff, and on confirmation (or `--yes`) rewrites the stored list.

- Installed something → it appears in the list.
- Removed something → it disappears from the list. No `state: absent` tombstones are needed, because restore always targets a **fresh** install: anything not listed simply never gets installed.
- Each capture is one git commit, which gives a readable change log as a side effect. Footprint doesn't treat old commits as restore points.
- `ignore:` patterns per collector suppress noise (auto-installed drivers, transient snaps).

## 5. Sidecar user data (proposed)

- Synced with `rclone sync` to `<target>/<base>/user-data/`: a mirror, so deletions propagate. This is a single rolling copy. Anything deleted locally is deleted from the target at the next push.
- Option: `--backup-dir` on the rclone side to keep a short-lived trash of deleted files (safety net, not restore points).
- Size guard: `push` warns if the sidecar exceeds a configurable size, since a large sidecar means the machine is not really ephemeral.

## 6. Secrets (proposed)

- SSH keys, GPG keys, tokens on an explicit include list are packed into a tar, encrypted with **age**, and pushed to `<target>/<base>/secrets/`. They never go into git.
- The age identity (or passphrase) is kept **outside** the system: a password manager or paper. Recovery needs exactly two things the dead machine can't provide: target credentials and the age key.
- Targets can additionally be rclone `crypt` remotes, so config and sidecar are also encrypted at rest.

## 7. Hardware independence (decided goal)

Never captured: disk UUIDs, `fstab`, partition layout, GPU or driver specifics, hostname-bound network config, MAC addresses. Hardware-sensitive needs are expressed as **detected roles** at restore time (e.g. "if an NVIDIA GPU is present, install the recommended driver"). The only requirement on new hardware is enough storage.

## 8. Replication and 3-2-1 (decided goal)

- `push` syncs to every target listed for the machine. Each target is reported separately; one failing target doesn't block the others.
- `status` flags a machine as **under-protected** if it has fewer than `min_targets` healthy targets (reachable, last push within N days) or no off-site target.
- Targets are independent copies written from the machine, not replicated from each other, so one corrupted target can't poison the rest.

## 9. Restore path (proposed)

On a new machine with a fresh install of the recorded distro:

```
# 1. bootstrap (one line, installs uv + footprint)
curl -fsSL <release-url>/install.sh | sh
# 2. restore
footprint restore --machine laptop --from wasabi
```

`restore` then:

1. Asks for target credentials (or reads an rclone config the user provides) and the age key.
2. Pulls `config/` (git bundle) and restores the footprint repo.
3. Decrypts `secrets/` (SSH keys first, so private repo clones work).
4. Runs `ansible-playbook -c local` for `common/` + `machines/<name>/`.
5. Runs `rclone copy` of `user-data/` back into place.
6. Prints a report: what was applied, what failed, what needs manual action (e.g. logins to cloud apps).

## 10. Proving it works (proposed)

`footprint test` replays a machine's footprint into a disposable container or VM (Incus/LXD, or a cloud VM) and reports failures. A backup that's never been restored is unverified. Run it after large changes, and never on the live machine.

## 11. Implementation (proposed)

- Python, packaged with `uv` (`uv tool install footprint`).
- Shells out to `rclone`, `ansible-playbook` and `age`; doesn't reimplement them.
- Collectors are plugins (one module each) so new package managers or desktops can be added.
- Ubuntu/Debian first; other distros via collector variants.

## 12. Open questions

- Restore onto a newer distro release than the one captured: allow with warnings, or require a matching release?
- Should `capture` run on a timer (systemd user timer), or only by hand followed by `push`?
- How to handle apps whose state lives in a local database (browsers, password managers). Out of scope as "cloud-synced already", or optional sidecar entries?
- Interaction with Enroll for the first capture: import its output, or write our own first-run collectors only?

## 13. Lineage: earlier experiments folded in (2026-11-14)

Two earlier repos tried versions of this idea. Both were deleted on 2026-11-14 once their useful parts were folded into Footprint.

**Ubuntu-Ansible-0425 (private, April 2025).** One-way desktop → laptop sync with Ansible: `capture-state`, `push-changes`, `apply-changes` playbooks. Its roles were never written. Kept: its scope list (apt, PPAs/sources, npm globals, Python packages, conda environments, dotfiles), now recipe-ubuntu.md layers 2, 3, 5 and 9.

**Claude-OS-Playbook (public template, March 2026).** Claude Code `/setup`, `/verify` and `/export` commands with eight subagents (system-packages, languages, cli-tools, containers, shell-config, claude-setup, desktop-apps, services) provisioning from a hand-written `profiles/*.yaml`.

- Kept: its category split (it matches the recipe layers); runtime managers (nvm, rustup, Go, SDKMAN); AI/agent tooling as a layer; marker-delimited managed blocks in dotfiles (`# --- <tool> START: <section> ---` … `END`) so edits are idempotent and never overwrite a user's file; "check what is present before installing, report what changed".
- Rejected, and why: **agent-driven provisioning**. Having an LLM interpret the profile on every run makes restore non-deterministic and dependent on an AI service being reachable and behaving the same way twice. That is the opposite of what a disaster-recovery path needs. Footprint keeps the agent only for one-time profiling (profiler.md); restore is plain Ansible.
- Rejected: **hand-written profiles as the source of truth**. They drift from the real machine the day after they're written. Footprint's playbook is captured from the machine, not authored.
- Different use, not covered: provisioning a *new* machine to a chosen template (as opposed to rebuilding a lost one). Partly reachable by restoring only `common/` onto a new machine; not a goal.
