# Cloudflare Tunnel - External Access

## Files

- `docker-compose.yml` - Cloudflared container configuration

## Architecture

```
Internet → Cloudflare Tunnel → Titan (192.168.10.80)
                                    ↓
                              cloudflared (Docker)
                                    ↓
                              localhost:9000 → Authentik
                              localhost:8082 → Trilium
```

## Services Exposed

| Service | Hostname | Internal |
|---------|----------|----------|
| Authentik | auth.harker.systems | localhost:9000 |
| Trilium | notes.harker.systems | localhost:8082 |

## Configuration

Routes configured in Cloudflare Dashboard:
1. Go to dash.cloudflare.com → Networking → Tunnels
2. Select tunnel → Routes
3. Add route → Published application
4. Enter hostname and service URL (e.g., `http://localhost:9000`)

## Notes

- Tunnel token stored in docker-compose.yml (rotate if compromised)
- Network mode: `host` to reach all localhost services
- No WARP/VPN required for public access