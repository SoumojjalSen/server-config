# Server Config

Deployment config for Oracle Cloud ARM VM (140.238.229.137).

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Oracle VM (140.238.229.137)                                 │
│                                                             │
│  Caddy (:80, :8080) ← only public-facing ports             │
│  ├── :80  /ai-toolbox/* → ai-toolbox (:3000)               │
│  ├── :80  /health       → "ok"                              │
│  └── :8080              → n8n (:5678)                       │
│                                                             │
│  ai-toolbox (:3000) ← MCP + AI gateway                     │
│  ├── /ai          → sends prompt to CLIProxyAPI             │
│  ├── /mcp/:server → connects to Groww/Kite MCP servers      │
│  └── /skills      → lists available AI skills               │
│                                                             │
│  CLIProxyAPI (:8317) ← proxies Claude Team subscription     │
│                                                             │
│  n8n (:5678) ← workflow scheduler + UI                      │
│                                                             │
│  Volumes:                                                   │
│  ├── cliproxyapi_data  → /CLIProxyAPI (config.yaml)         │
│  ├── cliproxyapi_auth  → /root/.cli-proxy-api (Claude auth) │
│  ├── mcp_auth          → /root/.mcp-auth (Groww auth)       │
│  ├── n8n_data          → /home/node/.n8n (workflows)        │
│  └── caddy_data        → /data (TLS certs, future HTTPS)    │
│                                                             │
│  Config files (SCP'd from GitHub on deploy):                │
│  ├── ~/apps/server-config/docker-compose.yml                │
│  └── ~/apps/server-config/Caddyfile                         │
│                                                             │
│  Env files (created manually, has secrets):                  │
│  └── /etc/ai-toolbox/.env                                   │
└─────────────────────────────────────────────────────────────┘
```

## Two CI/CD Flows

There are two separate repos with two separate purposes:

### Flow A: ai-toolbox repo → builds Docker image

```
You push code (index.js, skills, Dockerfile)
    │
    ▼
GitHub Actions CI:
    1. Reads the Dockerfile
    2. Builds a Docker image (your code + node_modules baked in)
    3. Pushes image to ghcr.io/soumojjalsen/ai-toolbox:latest
    │
    ▼
Image stored on GitHub Container Registry. Nothing happens on VM yet.
```

### Flow B: server-config repo → deploys to VM

```
You push config (docker-compose.yml, Caddyfile)
    │
    ▼
GitHub Actions deploy:
    1. actions/checkout → downloads repo on GitHub's temporary runner
    2. scp-action → copies docker-compose.yml + Caddyfile to VM
       (uses SSH key from GitHub Secrets)
    3. ssh-action → SSHs into VM and runs:
       - docker-compose down (stop old containers)
       - docker-compose pull (pulls latest images from ghcr.io)
       - docker-compose up -d (starts containers)
    │
    ▼
VM now runs latest config + latest images.
```

### Why SCP, not curl?

Originally the deploy script used `curl` to download files from `raw.githubusercontent.com`. But GitHub's CDN caches raw files for ~5 minutes. The deploy runs immediately after push — so it would download the **old** cached file, not the one you just pushed.

```
Before (broken):
  push → GitHub CDN (cached ~5 min) → curl on VM → stale file → wrong config deployed

After (fixed):
  push → actions/checkout (downloads from git, not CDN) → SCP to VM → always latest
```

SCP copies files directly from the GitHub runner (which has the exact commit you pushed) to the VM. No CDN, no caching, always fresh.

### How the two flows connect

```
ai-toolbox repo                    server-config repo
      │                                   │
      ▼                                   ▼
GitHub Actions CI                  GitHub Actions deploy
      │                                   │
      ▼                                   │
Pushes IMAGE to ghcr.io            SCPs CONFIG to VM
      │                                   │
      └──────────┐     ┌──────────────────┘
                 ▼     ▼
            Oracle VM runs:
            docker-compose pull  ← pulls IMAGE from ghcr.io
            docker-compose up -d ← reads CONFIG from ~/apps/server-config/
```

### Where do the SSH keys come from?

GitHub Secrets (repo Settings → Secrets → Actions):

| Secret | Value | Used by |
|--------|-------|---------|
| `SSH_PRIVATE_KEY` | Your Oracle VM SSH private key | scp-action and ssh-action to authenticate |
| `VM_HOST` | `140.238.229.137` | Where to connect |
| `VM_USER` | `ubuntu` | SSH username |

These are set once. GitHub Actions reads them at runtime — they never appear in logs or code.

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

## What lives where

| What | Where | How it gets there |
|------|-------|-------------------|
| App code (index.js, skills) | Inside Docker image | Built by ai-toolbox CI, pulled via `docker-compose pull` |
| docker-compose.yml | `~/apps/server-config/` on VM | SCP'd by deploy workflow |
| Caddyfile | `~/apps/server-config/` on VM | SCP'd by deploy workflow |
| Secrets (.env) | `/etc/ai-toolbox/.env` on VM | Created manually once, never in git |
| Auth tokens | Docker volumes on VM | Created by one-time login, persisted in volumes |
| n8n workflows | Docker volume on VM | Created in n8n UI, persisted in n8n_data volume |

## Logs

```bash
docker logs cliproxyapi --tail 20   # Claude proxy
docker logs ai-toolbox --tail 20    # MCP + AI gateway
docker logs n8n --tail 20           # Workflow engine
docker logs caddy --tail 20         # Reverse proxy
docker-compose logs --tail 10       # All at once
docker-compose logs -f              # Live follow all
```

## Manual commands (on VM)

```bash
cd ~/apps/server-config
docker-compose ps                   # Status
docker-compose pull                 # Pull latest images
docker-compose up -d                # Start all
docker-compose down                 # Stop all
docker-compose restart ai-toolbox   # Restart one
```
