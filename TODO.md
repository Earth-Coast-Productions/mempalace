# mempalace — open follow-ups

## Done

- [x] **Disable the idle-exit watchdog that bricked SSE tool-fetch (`MEMPALACE_MCP_IDLE_HOURS=0`).** 2026-06-09. mempalace's server self-exits after 8h idle "to release file handles" (#1552) — a workaround for *ephemeral per-session stdio* servers leaking ChromaDB/HNSW handles on Windows. This deployment is one long-lived Linux service behind mcp-proxy; when the child idle-exited, **mcp-proxy (0.11 and 0.12) did not respawn it**, so the next SSE client got a proxy-handled `initialize` OK but `tools/list` forwarded to the dead child returned JSON-RPC `{code:0,""}` → clients showed *"Reconnected to mempalace, but fetching tools failed: MCP error 0"*. Recurred every 8h-idle window; only a manual `systemctl restart` fixed it (until the next window). Set `Environment=MEMPALACE_MCP_IDLE_HOURS=0` in the unit (passed to the child via `--pass-environment`); the watchdog now never starts. Verified `tools/list` returns 30 tools over both Caddy:8111 and mcp-proxy:8112. Also bumped mcp-proxy 0.11.0→0.12.0 (OAuth shim still serves JSON 404s; no behavior change observed).
- [x] **Reapply patches on every service start (ExecStartPre hardening).** 2026-06-09. The unit had no patch-reapply step — patches were only reapplied by the weekly cron, so a manual `pip install` + restart outside that cron would start an *unpatched* server and reintroduce `diary_write`'s top-level `anyOf` (HTTP 400, bricks every connecting client). Factored reapply into `scripts/mempalace-apply-patches` (single source of truth; idempotent), wired it as `mempalace.service` `ExecStartPre=+` (root, fail-closed), and pointed `mempalace-auto-update` at the same script. Activates on next service start.
- [x] **Audit all mempalace tools for top-level `oneOf`/`allOf`/`anyOf`.** 2026-06-09. Introspected the live stdio server's `tools/list` out-of-band — all 30 tools are clean `type: object`; `diary_write` was the only offender and the patch removes it. Re-run the introspection (not a source grep — the schema is generated) after each upgrade; recipe in `/opt/standards/DEBUGGING.md`.

## Watching upstream

- [ ] **Retire `patches/0001-diary_write-remove-toplevel-anyOf.patch` when MemPalace/mempalace#1728 merges.** The Sunday auto-update will halt with "patch does not apply cleanly" once upstream's fix lands — that's the trigger. Delete the `.patch` file, commit, re-run the script manually to confirm.
  - Upstream issues: [#1728](https://github.com/MemPalace/mempalace/issues/1728), [#1711](https://github.com/MemPalace/mempalace/issues/1711)
  - The +1 comment with the workaround pattern is at https://github.com/MemPalace/mempalace/issues/1728#issuecomment-4655006361

- [ ] **Watch mcp-proxy for inbound OAuth-discovery handling.** If a future release adds a path-rewriter or canned-response option for the well-known URLs, the Caddy shim could be retired in favor of in-proxy handling. Not blocking; the Caddy layer is small and stable.

- [ ] **Residual fragility: mcp-proxy does not respawn (or notice) a dead stdio child.** The idle-exit fix removes the only *observed* cause of child death, but if the child ever dies for another reason (crash, OOM), mcp-proxy keeps serving `initialize` while `tools/list` fails silently — the same `MCP error 0` symptom, no service crash to trigger `Restart=on-failure`. Not building a healthcheck/watchdog now (conservative scope; trigger eliminated), but if `MCP error 0` ever reappears without an idle-exit log line, the fix is a periodic `tools/list` probe (systemd timer) that restarts mempalace on failure. Upstream candidate: mcp-proxy child-supervision/respawn.

## Possible improvements

- [ ] **Add a CI check that validates Caddyfile syntax.** `caddy validate --config caddy/Caddyfile --adapter caddyfile` as a pre-commit or Forgejo action — catches typos before they hit the live config.

- [ ] **Add a CI check that dry-runs each patch against the latest pinned mempalace version.** Catches "patch broken by upstream refactor" before the Sunday auto-update halts production.

- [ ] **Re-audit tool schemas for top-level `oneOf`/`allOf`/`anyOf` after each upgrade.** `diary_write` was the only offender in 3.4.0 (see Done above), but a future version could introduce another. The symptom is silent until an SDK sends the request, then it bricks the session — so check proactively via the out-of-band `tools/list` introspection (the schema is generated, so a source grep misses it). Recipe in `/opt/standards/DEBUGGING.md`. Good candidate to fold into the patch-dry-run CI check above.

## Setup gaps (low priority)

- [ ] **The `mempalace` user / ZFS dataset / venv install steps live in `/opt/standards/ENVIRONMENT.md`, not in this repo.** Bare-metal restore documents the clone-and-cp flow but a future "set up mempalace on a brand-new host" runbook could live here as `docs/bootstrap.md`. Decide whether to import or keep cross-referenced.
