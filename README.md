# mempalace

Deployment configs for the [MemPalace](https://github.com/MemPalace/mempalace) MCP server running on `ecp-server` (192.168.33.10).

The mempalace package itself is installed from PyPI into a venv at `/opt/mempalace/venv` and is auto-updated weekly by the script in `scripts/`. This repo tracks only the deployment glue — systemd unit, reverse-proxy shim, and the update wrapper.

## Architecture

```
                                ┌── OAuth-discovery + /register ──┐
                                │   (returns JSON 404)             │
   Claude Code MCP clients ──→ Caddy :8111 (LAN)                    │
   (host, fleet VMs, Signal)    │                                   │
                                └── everything else ──→ mcp-proxy ──┘
                                       127.0.0.1:8112
                                            ↓ (stdio)
                                   python -m mempalace.mcp_server
                                            ↓
                                   /opt/mempalace/data/palace (chroma)
```

| Component | Bind | Purpose |
|---|---|---|
| Caddy (`/etc/caddy/Caddyfile`) | `192.168.33.10:8111` | OAuth-probe shim + reverse proxy |
| mcp-proxy 0.12.0 (`/opt/mempalace/venv/bin/mcp-proxy`) | `127.0.0.1:8112` | SSE ↔ stdio bridge for the MCP server |
| mempalace server | stdio (child of mcp-proxy) | The actual MCP tool implementation |

Clients connect to `http://192.168.33.10:8111/sse` exactly as before — the shim is transparent to them.

## Why the Caddy shim exists

Newer Claude Code MCP SDK probes six OAuth-discovery paths plus `POST /register` on every connect/reconnect, expecting JSON error bodies per RFC 6749 §5.2. `mcp-proxy` 0.11.0 returns plain-text "Not Found" 404s for these paths, which the SDK can't parse — it then reports `SDK auth failed: Invalid OAuth error response: SyntaxError` and refuses to fetch tools. Caddy intercepts the probe paths and returns proper `application/json` 404 bodies so the SDK falls through to no-auth cleanly. Bug surfaced 2026-06-08; shim deployed 2026-06-09.

## Files in this repo

| Path | Live destination | Notes |
|---|---|---|
| `systemd/mempalace.service` | `/etc/systemd/system/mempalace.service` | Runs mcp-proxy bound to localhost:8112; `ExecStartPre` reapplies patches before every start; sets `MEMPALACE_MCP_IDLE_HOURS=0` to disable the idle-exit watchdog (see below) |
| `caddy/Caddyfile` | `/etc/caddy/Caddyfile` | Public LAN bind on .10:8111, OAuth shim, reverse-proxy to localhost:8112 |
| `scripts/mempalace-apply-patches` | `/usr/local/bin/mempalace-apply-patches` | Idempotent patch reapply — single source of truth; called by the unit's `ExecStartPre` and by the auto-update script |
| `scripts/mempalace-auto-update` | `/usr/local/bin/mempalace-auto-update` | Run weekly by the cron entry below |
| `configs/etc/cron.d/mempalace-auto-update` | `/etc/cron.d/mempalace-auto-update` | Sunday 04:00 UTC trigger for the auto-update |

## Ops

```bash
# Status
systemctl status mempalace
systemctl status caddy

# Logs
journalctl -u mempalace -f
journalctl -u caddy -f

# Restart (breaks live MCP sessions on the fleet until each /mcp reconnects — warn first)
systemctl restart mempalace

# Caddy config reload (does not affect live sessions)
systemctl reload caddy
```

### Deploying a config change from this repo

Live `/etc/` is the runtime source of truth; this repo is the declared state. Both directions are explicit, manual `cp`.

```bash
# After editing in repo:
sudo cp systemd/mempalace.service /etc/systemd/system/mempalace.service
sudo cp caddy/Caddyfile /etc/caddy/Caddyfile
sudo cp scripts/mempalace-apply-patches /usr/local/bin/mempalace-apply-patches
sudo cp scripts/mempalace-auto-update /usr/local/bin/mempalace-auto-update
sudo chmod +x /usr/local/bin/mempalace-apply-patches /usr/local/bin/mempalace-auto-update
sudo systemctl daemon-reload
sudo systemctl restart mempalace caddy
```

### Drift check

```bash
diff -q systemd/mempalace.service /etc/systemd/system/mempalace.service
diff -q caddy/Caddyfile /etc/caddy/Caddyfile
diff -q scripts/mempalace-apply-patches /usr/local/bin/mempalace-apply-patches
diff -q scripts/mempalace-auto-update /usr/local/bin/mempalace-auto-update
```

## Auto-update

`scripts/mempalace-auto-update` runs Sunday 04:00 UTC via the cron entry at `configs/etc/cron.d/mempalace-auto-update`. It snapshots `/opt/mempalace/data/palace`, runs `pip install --upgrade mempalace`, reapplies local patches from `patches/`, restarts the service, and rolls back on health-check failure. Keeps the 4 most recent snapshots.

The cron entry deliberately fires during the lowest-traffic window because each restart breaks long-lived MCP sessions on other VMs (-32602) until clients run `/mcp` to reconnect.

## Why the idle-exit watchdog is disabled

The unit sets `Environment=MEMPALACE_MCP_IDLE_HOURS=0`. mempalace's server has a watchdog (#1552) that exits the process after 8h with no requests, "to release ChromaDB/HNSW file handles" — designed for *ephemeral per-session stdio* servers on Windows that never self-terminate when a Claude Code session ends. This deployment is the opposite: one long-lived Linux service fronted by mcp-proxy. When the child idle-exited, **mcp-proxy did not respawn it**, so the next SSE client got a proxy-handled `initialize` OK but a `tools/list` forwarded to the dead child came back as JSON-RPC `{code:0,""}` — surfacing in Claude Code as *"Reconnected to mempalace, but fetching tools failed: MCP error 0"*. It recurred every 8h-idle window and was only cleared by a manual restart. Disabling the watchdog keeps the child alive for the life of the service. The env var reaches the child because mcp-proxy is launched with `--pass-environment`.

## Local patches

`patches/` holds unified diffs applied on top of each pip-installed mempalace version. Each patch documents an upstream incompatibility that hasn't shipped a fix yet. Reapplication lives in one place — `scripts/mempalace-apply-patches` — with **two callers**: the `mempalace.service` `ExecStartPre` (so *every* start reapplies first — a manual restart after a stray `pip` reinstall can't bring up an unpatched server) and `mempalace-auto-update` (after each `pip install --upgrade`). The reapply:

1. Skips patches already applied (reverse-patch dry-run).
2. Aborts if a patch no longer applies cleanly (likely upstream merged the fix or refactored the target — retire the patch manually). As `ExecStartPre`, a non-zero exit aborts service startup *on purpose* — a service that's down is recoverable, but a server started unpatched bricks every client that connects (the bad `diary_write` schema rides in `tools[]` on every request → HTTP 400 → the client can't even reconnect to pick up a fix; recovery is a brand-new session).
3. Applies the rest in lexical order.

Current patches:

| Patch | Target | Reason |
|---|---|---|
| `0001-diary_write-remove-toplevel-anyOf.patch` | `mcp_server.py` | Anthropic's Messages API rejects tools with `oneOf`/`allOf`/`anyOf` at `input_schema` root (HTTP 400). Server already enforces the constraint at handler time. Upstream: [MemPalace/mempalace#1728](https://github.com/MemPalace/mempalace/issues/1728) (also [#1711](https://github.com/MemPalace/mempalace/issues/1711)). |

### Adding a patch

```bash
# 1. Make the local fix in /opt/mempalace/venv/lib/python3.12/site-packages/mempalace/
# 2. Diff against the pristine wheel:
mkdir -p /tmp/pristine && cd /tmp/pristine
/opt/mempalace/venv/bin/pip download --no-deps mempalace==<VERSION> -d .
unzip -o mempalace-<VERSION>-*.whl 'mempalace/<file>.py'
diff -u mempalace/<file>.py /opt/mempalace/venv/lib/python3.12/site-packages/mempalace/<file>.py \
    | sed -E '1s|.*|--- a/<file>.py|; 2s|.*|+++ b/<file>.py|' \
    > /root/mempalace/patches/NNNN-description.patch
# 3. Verify it applies cleanly to the pristine + matches live:
cd /tmp/pristine/mempalace && patch -p1 --dry-run < /root/mempalace/patches/NNNN-*.patch
# 4. Commit + push.
```

### Retiring a patch

When upstream merges the fix in version X, the auto-update will halt with "patch does not apply cleanly". Delete the `.patch` file, commit, then re-run the script.

## Related

- Upstream: https://github.com/MemPalace/mempalace
- Host docs: `/root/ecp-server/CLAUDE.md` (Configuration Files Summary table includes mempalace entries)
- MCP transport: `mcp-proxy` https://github.com/sparfenyuk/mcp-proxy
