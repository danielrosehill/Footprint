# Recipe: Ubuntu (workstation, laptop, home server)

Status: draft recipe, 2026-11-14. This is the first **formula**: the Footprint recipe for one OS family. Other families get their own formula (see the end). It isn't a fixed template. It lists the **layers** every Ubuntu machine has. Which items in each layer belong in a machine's footprint is decided per machine by the profiler (see [profiler.md](profiler.md)).

## Why not one template

A real sample machine (Ubuntu 26.04 laptop, KDE, profiled 2026-11-14) had:

| Layer | Count |
|---|---|
| apt packages marked manual | 673 |
| third-party apt sources | 32 |
| flatpak apps | 22 |
| snaps | 23 |
| `uv tool` installs | 21 |
| pipx installs | 32 |
| global npm packages | 11 |
| enabled systemd units (system / user) | 170 / 64 |
| timers | 24 |

Most of the 673 "manual" apt packages are distro seed or long-forgotten experiments, and most of the 170 units are distro defaults. Deciding what is intent and what is noise takes judgement, so it is done once, interactively, by the profiler. After that, capture is mechanical.

## Layers

Each layer is a collector (live state → sorted list) plus an Ansible role (list → installed state). Order matters on restore: each layer depends on the ones above it.

| # | Layer | Captured from | Restored with | Notes |
|---|---|---|---|---|
| 0 | Base | `/etc/os-release` (release, arch), desktop environment | precondition check only | The user installs the base OS by hand. Footprint refuses or warns on a release mismatch. |
| 1 | User | username, uid, shell, groups | `user`, `group` | groups such as `docker`, `dialout`, `libvirt` |
| 2 | Package sources | `/etc/apt/sources.list.d/*`, `/etc/apt/keyrings`, `/usr/share/keyrings` (non-distro), PPAs | `deb822_repository`, `get_url` for keys | Record key **URLs** where known, so keys are fetched fresh rather than copied stale. |
| 3 | System packages | `apt-mark showmanual` minus the release's seed manifest, minus the ignore list | `apt` | Also `.deb` files installed from URLs (source URL recorded by profiler). |
| 4 | Sandboxed apps | `flatpak list --app` + remotes; `snap list` (non-base, with channel/confinement) | `community.general.flatpak`, `snap` | |
| 5 | Language toolchains | `uv tool list`, `uv python list --only-installed`, `pipx list --json`, `pip list --user`, `conda env list` + `conda env export --from-history` per env, `npm ls -g`, `~/.cargo/.crates2.json`, `go` binaries, nvm/SDKMAN versions | `command` with `creates:`, conda `env create -f` | Conda envs are stored as `environment.yml` files in the footprint repo. |
| 6 | Loose binaries | AppImages, `~/.local/bin`, `/usr/local/bin`, `/opt/*` | `get_url` from a recorded source URL | The profiler asks where each came from. Unknown origin goes to the sidecar. |
| 7 | Repositories | git checkouts under configured roots: path, remote, branch | `git` clone | Never content. Unpushed or remote-less repos are flagged in `drift`. |
| 8 | System config | conffiles changed from package default (`dpkg --verify`), plus an include list under `/etc` | `copy` | e.g. sudoers drop-ins, sysctl, udev rules, NetworkManager *non-secret* settings |
| 9 | User config | dotfiles on an include list | `copy` / `template` | If a dotfile manager (yadm, chezmoi, stow) is already in use, Footprint records "clone dotfiles repo X with tool Y" instead of copying files. |
| 10 | Desktop settings | KDE rc files on an include list, or `dconf dump /` for GNOME | `copy` / `dconf load` | Workstation and laptop only. |
| 11 | Services and schedules | enabled units and timers that differ from distro defaults, user crontab, user units in `~/.config/systemd/user` | `systemd_service`, `copy`, `cron` | |
| 12 | Containers | compose files and the repos they live in, named images | treated as repositories (layer 7) plus `docker compose up` | **Volumes are data, not footprint.** See the home server variant. |
| 13 | Secrets | include list: `~/.ssh`, `~/.gnupg`, tokens | age-encrypted tar, not Ansible | Restored first, before layer 7, so private clones work. |
| 14 | Sidecar | include list of local-only data | `rclone sync` | |

## Variants

The layers are the same across all three. The variants differ in emphasis.

**Workstation (desktop).** Heaviest on layers 3–6 and 10. Hardware roles are detected at restore time (GPU vendor → driver, multi-disk storage is out of scope: Footprint restores onto whatever root disk you give it). Big local data (RAID arrays, media) is outside the ephemeral principle and needs a conventional backup.

**Laptop.** Same as workstation plus power and input settings. Wi-Fi profiles hold passwords, so they go in secrets, not config.

**Home server.** Mostly layers 2, 3, 7, 11, 12. The footprint rebuilds the *host*: packages, Docker, compose repos, units. Docker **volumes and databases are state**, and a server's state usually breaks the ephemeral principle. The recipe's rule: if a service's data can't be rebuilt from upstream, it needs its own backup (database dump, volume backup) and Footprint records *where that backup lives and how to restore it* as a manual step in the restore report. Footprint doesn't try to be a database backup tool.

## Other formulas (later)

| Family | Approach |
|---|---|
| Raspberry Pi OS / Debian (arm64) | Same layers as Ubuntu. Different seed manifest, no snap. |
| OPNsense | Not Ansible-rebuilt. The whole config is one `config.xml`, so the formula is "export `config.xml` to the targets on change" plus a plugin list. |
| Synology DSM | Config export (`.dss`) plus a package list. User data on the NAS is the backup target, not something Footprint captures. |

## What the old experiment scoped (folded in here)

The 2025 Ubuntu Ansible experiment scoped: Debian packages, third-party sources and PPAs, global npm packages, Python packages, conda environments and dotfiles. All six are covered above: layers 2, 3, 5 (npm, Python, conda) and 9. That repo was a skeleton (playbooks referencing roles that were never written) and has been retired in favour of this one.
