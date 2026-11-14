# Footprint

**A rolling footprint of your system state, kept as an Ansible playbook, as your disaster-recovery backup.**

Status: concept only (2026-11-14). No code yet. This repo records the idea and the intended design.

## The principle: the computer is ephemeral

Footprint rests on one idea: **a user's computer is ephemeral.** Almost nothing on it should be local-only. Code lives in GitHub, documents in a cloud drive, mail and calendars in their services, secrets in a secrets manager. Every important part of the system is synced somewhere else.

We already treat development environments this way. Nobody backs up a Python virtualenv or `node_modules`. You keep a `requirements.txt` or `package.json` and rebuild the environment when you need it. **Footprint is `requirements.txt` for the whole filesystem.** It records what the machine is (user, packages, config, cloned repos, tools, services) so the machine can be rebuilt, not copied. The small amount of genuinely local data travels alongside it as a **sidecar**.

**When the machine dies, you don't restore an image. You reinstall the base OS and replay the footprint.**

### Who this is for

- **Good fit:** people who already work cloud-first, whose computer is close to a thin client, and who want a new machine back to "theirs" in one command.
- **Poor fit:** people who keep large or important data only on their local disk and don't sync it anywhere. Footprint's sidecar handles a modest set of loose files, not a full local archive. Use a conventional backup tool for that.

### What it is (and isn't)

| Footprint is | Footprint is not |
|---|---|
| A **disaster-recovery** utility: a one-time full restore onto a fresh install | A rollback, undo or restore-point tool |
| A CLI you run on the machine (manually or on a timer) | An agent skill or AI workflow |
| A single rolling copy of system state, updated incrementally | A version history or snapshot archive |
| Hardware-independent: restores onto any machine with enough storage | A disk imager or partition cloner |
| An overlay on a normal distro | An operating system (it is not NixOS) |

## How it works

1. **Capture.** Footprint inspects the live machine and updates a per-machine Ansible playbook. Install a package and it's added. Remove one and it's dropped. The playbook follows the machine; nobody writes it by hand.
2. **Sidecar.** Loose user data that the playbook can't regenerate (chosen dotfiles, app data, local-only folders) is synced as a single rolling copy.
3. **Replicate.** The playbook, sidecar and an encrypted secrets blob go to **at least two backup targets**, never just one.
4. **Restore.** On new hardware: install the base distro, run `footprint restore`, enter your target credentials and secrets key, and wait.

## Federation: one footprint, many machines, many targets

```
footprint repo (private git, e.g. GitHub)
├── footprint.yml              # targets + machines registry
├── common/                    # roles shared by every machine (user, shell, base tools)
└── machines/
    ├── laptop/                # this machine's playbook + captured state
    └── desktop/

each backup target (cloud bucket, NAS, ...)
└── <machine-base>/            # e.g. laptop/
    ├── config/                # the footprint repo (git bundle)
    ├── user-data/             # sidecar: loose user data, single rolling copy
    └── secrets/               # encrypted blob (SSH keys, tokens); never in git
```

- **Repo → machine → targets → machine base → `config/` + `user-data/`.**
- Shared roles live in `common/`, so something you want everywhere is declared once.
- Several machines can share a target, each under its own machine base.
- Restore depends only on a target, never on GitHub being reachable.

## 3-2-1 by default

Every machine's footprint is replicated to **at least two targets**, for example two cloud providers, or a NAS plus a cloud bucket. With the live machine, that gives three copies on two kinds of storage with at least one off-site.

- `footprint push` writes to every target and reports each one separately.
- A machine with fewer than two healthy targets is flagged as **under-protected** in `footprint status`, and `push` warns.
- Any single target is enough to restore.

## Built on existing tools, not reinvented

Footprint is a thin layer that adds the capture, registry, 3-2-1 and restore logic. Proven tools do the heavy lifting:

| Job | Tool | Why |
|---|---|---|
| Moving data to targets | [rclone](https://rclone.org/) | 70+ backends (S3, Wasabi, B2, SFTP/SMB/WebDAV for NAS boxes). `sync` gives the single-rolling-copy model natively. `crypt` gives client-side encryption. |
| Applying the footprint on restore | [Ansible](https://docs.ansible.com/) (`ansible-core`, local connection) | Idempotent, readable, already the standard. The playbook stays usable without Footprint. |
| Encrypting secrets | [age](https://age-encryption.org/) | Single key file or passphrase, easy to keep offline. |
| Optional first capture | [Enroll](https://hn.svelte.dev/item/46449852) | Can bootstrap a first playbook from an existing machine; Footprint then curates it. |

Restic was considered for data transfer and rejected as the default. Its snapshot model is the opposite of a single rolling copy, and the snapshot history and pruning add overhead we don't want. It could be added later as an alternative backend.

See [docs/design.md](docs/design.md) for the registry format, what gets captured, how removals work, the secrets model, hardware independence and the restore path.

## Intended commands (draft)

```
footprint init [--machine NAME]             # register this machine; first capture
footprint target add NAME <rclone-remote:path>   # add a backup target
footprint capture                           # compare live system with footprint, show diff, update playbook
footprint push [--target NAME]              # commit, then sync config + user-data + secrets to every target
footprint drift                             # report live-vs-footprint differences, change nothing
footprint status                            # per machine, per target: last push, size, reachable, 3-2-1 OK?
footprint restore --machine NAME --from <target>   # fresh install: pull, decrypt, run playbook, restore sidecar
footprint test --machine NAME               # restore into a throwaway container/VM to prove it rebuilds
```

## Prior art

Checked 2026-11-14. Nothing found does the incremental, rolling part:

- [Enroll](https://hn.svelte.dev/item/46449852) reverse-engineers a machine into Ansible roles. A one-off harvest, very noisy on desktops (around 1,000 roles reported).
- [ansible-clone](https://awesome.ecosyste.ms/api/v1/projects/github.com%2Fjsf0%2Fansible-clone) and [Replibook](https://hn.jakechampion.name/reader/https://github.com/AlexRosbach/Replibook) scan packages and services into a playbook. Also one-off.
- [chezmoi](https://www.chezmoi.io/) manages dotfiles, not system state.
- [Falconf](https://docs.rs/crate/falconf/0.3.0) tracks changes on top of a base install. Imperative, not Ansible.
- NixOS / Home Manager are declarative but replace how you run the system, which Footprint deliberately does not.

## Licence

MIT.
