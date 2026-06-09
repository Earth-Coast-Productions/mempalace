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
| mcp-proxy (`/opt/mempalace/venv/bin/mcp-proxy`) | `127.0.0.1:8112` | SSE ↔ stdio bridge for the MCP server |
| mempalace server | stdio (child of mcp-proxy) | The actual MCP tool implementation |

Clients connect to `http://192.168.33.10:8111/sse` exactly as before — the shim is transparent to them.

## Why the Caddy shim exists

Newer Claude Code MCP SDK probes six OAuth-discovery paths plus `POST /register` on every connect/reconnect, expecting JSON error bodies per RFC 6749 §5.2. `mcp-proxy` 0.11.0 returns plain-text "Not Found" 404s for these paths, which the SDK can't parse — it then reports `SDK auth failed: Invalid OAuth error response: SyntaxError` and refuses to fetch tools. Caddy intercepts the probe paths and returns proper `application/json` 404 bodies so the SDK falls through to no-auth cleanly. Bug surfaced 2026-06-08; shim deployed 2026-06-09.

## Files in this repo

| Path | Live destination | Notes |
|---|---|---|
| `systemd/mempalace.service` | `/etc/systemd/system/mempalace.service` | Runs mcp-proxy bound to localhost:8112 |
| `caddy/Caddyfile` | `/etc/caddy/Caddyfile` | Public LAN bind on .10:8111, OAuth shim, reverse-proxy to localhost:8112 |
| `scripts/mempalace-auto-update` | `/usr/local/bin/mempalace-auto-update` | Sunday 04:00 UTC cron entry at `/etc/cron.d/mempalace-auto-update` (lives in `ecp-server` repo) |

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
sudo systemctl daemon-reload
sudo systemctl restart mempalace caddy
```

### Drift check

```bash
diff -q systemd/mempalace.service /etc/systemd/system/mempalace.service
diff -q caddy/Caddyfile /etc/caddy/Caddyfile
diff -q scripts/mempalace-auto-update /usr/local/bin/mempalace-auto-update
```

## Auto-update

`scripts/mempalace-auto-update` runs Sunday 04:00 UTC via `/etc/cron.d/mempalace-auto-update` (tracked in the `ecp-server` repo). It snapshots `/opt/mempalace/data/palace`, runs `pip install --upgrade mempalace`, restarts the service, and rolls back on health-check failure. Keeps the 4 most recent snapshots.

The cron entry deliberately fires during the lowest-traffic window because each restart breaks long-lived MCP sessions on other VMs (-32602) until clients run `/mcp` to reconnect.

## Related

- Upstream: https://github.com/MemPalace/mempalace
- Host docs: `/root/ecp-server/CLAUDE.md` (Configuration Files Summary table includes mempalace entries)
- MCP transport: `mcp-proxy` https://github.com/sparfenyuk/mcp-proxy
