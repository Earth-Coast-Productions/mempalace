# Changelog

All notable changes to this repository are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This repo carries **no git tags and no `VERSION` file**, so entries are grouped
by date rather than by semantic version. (Version numbers that do appear below —
mcp-proxy 0.12.0, mempalace 3.4.1 — are the versions of *upstream* software this
repo deploys, not versions of this repo.)

Per the ECP TODO/CHANGELOG standard, `TODO.md` holds **open work only**;
completed work belongs here.

Entries below were derived from this repo's git history. The repo is small
(14 commits, 2026-06-09 → 2026-07-08) so its history is covered in full;
`git log` remains the authoritative record.

## [Unreleased]

### Changed
- **2026-11-20 — mcp-proxy 0.12.0 → 0.15.0** (mempalace 3.12.0 was already the latest on PyPI).
  Picks up the 0.13.1/0.13.2 security releases (cryptography 50, PyJWT 2.15, Starlette 1.10, certifi
  dropping the compromised emSign ECC root). Upgraded in place with pip's default only-if-needed strategy:
  12 packages moved (mcp 1.27→1.30 and the HTTP stack); **chromadb, numpy and onnxruntime untouched**, so
  the palace's on-disk format did not change. ⚠ **0.13.3 added Host/Origin validation**: behind Caddy, which
  forwards `Host: 192.168.33.10:8111`, an un-flagged 0.15 returns **421 Misdirected Request** to every LAN
  client. The unit now passes `--allowed-host=192.168.33.10:8111` (adds to the loopback defaults). Rehearsed
  on a palace copy first (hazard reproduced without the flag, fixed with it, foreign Host still 421), then
  verified live: real handshake through :8111 and :8112, OAuth shim JSON 404s, healthcheck green, and a
  fresh headless Claude Code `mempalace_status` = 5207 drawers. ⚠ Clients connected before the restart keep
  getting `404 Could not find session` until `/mcp` reconnect. Rollback: `/var/backups/mempalace-20261120-mcp-proxy-0.15/`
  (pip freeze + old unit — restore BOTH; 0.12 rejects `--allowed-host`) and ZFS `storage01/mempalace@pre-mcp-proxy-0.15`.

- **Git workflow: dual-push retired.** `CLAUDE.md` §Git workflow now shows a single push to
  `origin`; Forgejo push-mirrors to GitHub itself. The `github` remote, the `github.com-mempalace`
  alias and its deploy key (`SHA256:XaiaUlj7p23f62DJlS355tfCv6lWm3AlByqrOQF4584`) were removed and
  **shredded** on `.10`. Also records that **this repo is public on GitHub** while private in
  Forgejo — pre-existing, but worth knowing before committing anything sensitive.

Open follow-ups are tracked in `TODO.md`.

## 2026-07-08

### Added

- Self-heal and manual-fix runbook for palace outages (watchdog operation,
  Caddy gateway restart) in `CLAUDE.md`.
- Post-mortem of the 2026-07-07 Caddy-gateway outage, plus notes on the
  healthcheck watchdog and a Hindsight evaluation item, in the docs.

## 2026-07-07

### Added

- `mempalace-healthcheck` timer — a periodic real MCP handshake probe with
  self-heal escalation (`scripts/mempalace-healthcheck`,
  `systemd/mempalace-healthcheck.{service,timer}`).

## 2026-06-16

### Removed

- Retired `patches/0001-diary_write-remove-toplevel-anyOf.patch`. The
  `diary_write` top-level `anyOf` fix shipped upstream in mempalace 3.4.1, so
  the local patch was no longer needed.

## 2026-06-10

### Added

- Recorded post-idle-window verification of the idle-exit fix.
- Pinned upstream PR #1735 as the retirement trigger for patch 0001, noting
  it is *not* the same issue as the idle-exit fix.

## 2026-06-09

### Added

- Initial repo contents: systemd unit, Caddy OAuth-probe shim, and the
  auto-update wrapper — migrated out of the `ecp-server` repo.
- Tracking for the `diary_write` top-level `anyOf` patch and its idempotent
  reapply, with links to the upstream issues.
- Cron trigger for the weekly auto-update.
- `CLAUDE.md` (session instructions) and `TODO.md` (open follow-ups).

### Changed

- Patches are now reapplied on **every** service start via `ExecStartPre`,
  rather than only by the weekly auto-update cron.
- Bumped mcp-proxy to 0.12.0.

### Fixed

- Disabled the upstream idle-exit watchdog (`MEMPALACE_MCP_IDLE_HOURS=0`),
  which had been bricking SSE tool-fetch for this long-lived deployment.
