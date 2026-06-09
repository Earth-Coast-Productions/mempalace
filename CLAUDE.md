# mempalace — Claude session instructions

Deployment configs for the MemPalace MCP server running on ecp-server (192.168.33.10).

## Where things live

- **This repo** (`/root/mempalace`): systemd unit, Caddy shim, auto-update script + cron, patches against the pip-installed mempalace package. Forgejo `earthcoast/mempalace` is source of truth; GitHub `Earth-Coast-Productions/mempalace` is the mirror.
- **Live `/etc/`**: runtime source of truth. The repo holds the declared state. No automatic sync in either direction — `cp` from repo to live is explicit; updating from live → repo is also explicit, after the change has been validated in-place.
- **mempalace itself**: `/opt/mempalace/venv` (pip-installed package). Patches in this repo's `patches/` are reapplied after every `pip install --upgrade` by the auto-update script.

## Read these first

- `README.md` — architecture, ports, the why behind the Caddy shim, the patch workflow.
- `/opt/standards/ENVIRONMENT.md` — fleet-wide context for this deployment.
- `/opt/standards/DEBUGGING.md` "MCP server: tool fetch fails after reconnect" — diagnostic procedure if `/mcp` breaks.

## Git workflow

Forgejo first, GitHub mirror second. Always verify both refs match after pushing:

```bash
git push origin main
git push github main
git ls-remote origin refs/heads/main
git ls-remote github refs/heads/main
```

The github remote uses a per-repo SSH alias (`github.com-mempalace`) — see `~/.ssh/config`.

## Deploying a change

Live `/etc/` is the runtime source of truth. To deploy a tracked config change from this repo:

```bash
# After editing the file here:
sudo cp systemd/mempalace.service /etc/systemd/system/mempalace.service
sudo cp caddy/Caddyfile /etc/caddy/Caddyfile
sudo cp scripts/mempalace-apply-patches /usr/local/bin/mempalace-apply-patches
sudo cp scripts/mempalace-auto-update /usr/local/bin/mempalace-auto-update
sudo cp configs/etc/cron.d/mempalace-auto-update /etc/cron.d/mempalace-auto-update
sudo chmod +x /usr/local/bin/mempalace-apply-patches /usr/local/bin/mempalace-auto-update
sudo systemctl daemon-reload
sudo systemctl restart mempalace caddy        # warn fleet before restarting mempalace
```

Drift check:

```bash
diff -q systemd/mempalace.service /etc/systemd/system/mempalace.service
diff -q caddy/Caddyfile /etc/caddy/Caddyfile
diff -q scripts/mempalace-apply-patches /usr/local/bin/mempalace-apply-patches
diff -q scripts/mempalace-auto-update /usr/local/bin/mempalace-auto-update
diff -q configs/etc/cron.d/mempalace-auto-update /etc/cron.d/mempalace-auto-update
```

## Restart caveat

Restarting `mempalace.service` breaks long-lived MCP sessions on every other VM with JSON-RPC `-32602` until each client runs `/mcp` to reconnect — the stale SSE session survives the restart but never re-sends the MCP `initialize` handshake. **Always warn on Signal before restarting**, and remind operators to run `/mcp` afterwards. Reloading Caddy (`systemctl reload caddy`) does NOT have this side effect — it's hot-swappable.

## Adding a patch

Patches in `patches/` are unified diffs applied on top of each pip-installed mempalace version. Reapplication lives in one place — `scripts/mempalace-apply-patches` (installed to `/usr/local/bin/`) — with two callers: the auto-update script after each `pip install --upgrade`, and the `mempalace.service` `ExecStartPre` (`+`-prefixed, runs as root) so **every** service start reapplies first. That closes the gap where a manual restart after a stray `pip` reinstall could start an unpatched server — a non-zero exit from the reapply aborts startup on purpose (service-down is recoverable; an unpatched server bricks every client that connects). The reapply:

1. Skips patches whose reverse-patch dry-run succeeds (already applied — idempotent).
2. Aborts if a patch no longer applies cleanly (likely upstream merged the fix or refactored the target — retire it manually).
3. Applies the rest in lexical order.

To add a new patch, see the "Adding a patch" recipe in `README.md`.

## Where this repo isn't yet the source of truth

- The `mempalace` user account (uid 996), the ZFS dataset layout, and the venv install are not tracked — they're host-level bootstrap steps captured in `/opt/standards/ENVIRONMENT.md` setup section. If you're rebuilding mempalace from scratch on a fresh host, follow that section, then clone this repo and `cp` the configs into place.
