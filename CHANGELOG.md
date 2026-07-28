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
