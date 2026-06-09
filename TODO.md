# mempalace — open follow-ups

## Watching upstream

- [ ] **Retire `patches/0001-diary_write-remove-toplevel-anyOf.patch` when MemPalace/mempalace#1728 merges.** The Sunday auto-update will halt with "patch does not apply cleanly" once upstream's fix lands — that's the trigger. Delete the `.patch` file, commit, re-run the script manually to confirm.
  - Upstream issues: [#1728](https://github.com/MemPalace/mempalace/issues/1728), [#1711](https://github.com/MemPalace/mempalace/issues/1711)
  - The +1 comment with the workaround pattern is at https://github.com/MemPalace/mempalace/issues/1728#issuecomment-4655006361

- [ ] **Watch mcp-proxy for inbound OAuth-discovery handling.** If a future release adds a path-rewriter or canned-response option for the well-known URLs, the Caddy shim could be retired in favor of in-proxy handling. Not blocking; the Caddy layer is small and stable.

## Possible improvements

- [ ] **Add a CI check that validates Caddyfile syntax.** `caddy validate --config caddy/Caddyfile --adapter caddyfile` as a pre-commit or Forgejo action — catches typos before they hit the live config.

- [ ] **Add a CI check that dry-runs each patch against the latest pinned mempalace version.** Catches "patch broken by upstream refactor" before the Sunday auto-update halts production.

- [ ] **Audit whether other mempalace tools have top-level `oneOf`/`allOf`/`anyOf`.** Today only `diary_write` was the culprit, but the same JSON Schema pattern could exist elsewhere — the symptom is silent unless an SDK happens to send the request. `grep -rn "anyOf\|oneOf\|allOf" /opt/mempalace/venv/lib/python3.12/site-packages/mempalace/` after each upgrade.

## Setup gaps (low priority)

- [ ] **The `mempalace` user / ZFS dataset / venv install steps live in `/opt/standards/ENVIRONMENT.md`, not in this repo.** Bare-metal restore documents the clone-and-cp flow but a future "set up mempalace on a brand-new host" runbook could live here as `docs/bootstrap.md`. Decide whether to import or keep cross-referenced.
