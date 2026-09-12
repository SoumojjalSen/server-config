# Server Config

Deployment config for Oracle Cloud ARM VM (140.238.229.137).

## Services

| Port | Service | Access | What it does |
|------|---------|--------|-------------|
| 80 | Caddy | Public | Reverse proxy — routes `/ai-toolbox/*` to ai-toolbox |
| 8080 | Caddy → n8n | Public | Workflow UI |
| 3000 | ai-toolbox | Internal | MCP + AI gateway (`/ai`, `/mcp/*`, `/skills`, `/health`) |
| 5678 | n8n | Internal | Workflow engine (Caddy proxies via :8080) |
| 8317 | CLIProxyAPI | Internal | Claude Team proxy (OpenAI-compatible API) |

## URLs

- `http://140.238.229.137/ai-toolbox/health` — ai-toolbox health check
- `http://140.238.229.137/health` — Caddy health check
- `http://140.238.229.137:8080` — n8n UI

## Logs

```bash
docker logs cliproxyapi --tail 20   # Claude proxy
docker logs ai-toolbox --tail 20    # MCP + AI gateway
docker logs n8n --tail 20           # Workflow engine
docker logs caddy --tail 20         # Reverse proxy
docker-compose logs --tail 10       # All at once
docker-compose logs -f              # Live follow all
```

## Deploy

Push to this repo → GitHub Actions SCPs files to VM → restarts services.

## Manual commands (on VM)

```bash
cd ~/apps/server-config
docker-compose ps                   # Status
docker-compose pull                 # Pull latest images
docker-compose up -d                # Start all
docker-compose down                 # Stop all
docker-compose restart ai-toolbox   # Restart one
```
