# Footprint

Release tier: small (`v1.0`, `v1.1`, ...).

Status (2026-11-14): concept and design docs only, no code. The principle and the decided/proposed split are in `docs/design.md`; keep that file current as decisions are made.

Rules for this repo:
- It is a CLI utility, not an agent skill or plugin.
- Wrap existing tools (rclone, ansible-core, age); don't reimplement transfer, encryption or config application.
- Public repo: no personal machine names, hosts, buckets or credentials in examples. Use placeholders.
