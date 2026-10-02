# Server Config

Deployment config for Oracle Cloud ARM VM (140.238.229.137).

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Oracle VM (140.238.229.137)                                 │
│                                                             │
│  Public ports: 80 (Caddy health), 5678 (n8n UI)             │
│                                                             │
│  Caddy (:80) → /health "ok"                                 │
│                                                             │
│  ai-toolbox (127.0.0.1:8080, internal only — no auth)       │
│  ├── /ai          → Claude Code (claude -p) + web search    │
│  ├── /mcp/:server → connects to Groww/Kite MCP servers      │
│  └── /skills      → lists available AI skills               │
│                                                             │
│  n8n (:5678) ← workflow scheduler + UI, calls ai-toolbox    │
│                                                             │
│  Volumes:                                                   │
│  ├── mcp_auth          → /root/.mcp-auth (Groww auth)       │
│  ├── claude_sessions   → /root/.claude/projects (7 days)    │
│  ├── n8n_data          → /home/node/.n8n (workflows)        │
│  └── caddy_data        → /data (TLS certs, future HTTPS)    │
│                                                             │
│  Config files (SCP'd from GitHub on deploy):                │
│  ├── ~/apps/server-config/docker-compose.yml                │
│  └── ~/apps/server-config/Caddyfile                         │
│                                                             │
│  Env files (created manually, has secrets):                 │
│  └── /etc/ai-toolbox/.env (CLAUDE_CODE_OAUTH_TOKEN, ...)    │
└─────────────────────────────────────────────────────────────┘
```

## Two CI/CD Flows

There are two separate repos with two separate purposes:

### Flow A: ai-toolbox repo → builds Docker image

```
You push code (src/, skills, Dockerfile)
    │
    ▼
GitHub Actions CI:
    1. Reads the Dockerfile
    2. Builds a Docker image (your code + node_modules baked in)
    3. Pushes image to ghcr.io/soumojjalsen/ai-toolbox:latest
    4. SSHs into the VM: docker-compose pull ai-toolbox + up -d ai-toolbox
    │
    ▼
VM runs the new image within a few minutes of the push.
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
       - docker-compose pull (pulls latest images from ghcr.io)
       - docker-compose up -d --remove-orphans (recreates only changed containers)
       - docker-compose restart caddy (Caddyfile is a bind mount — needs a restart to load edits)
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
| 80 | Caddy | Public | Health check (`/health`) |
| 5678 | n8n | Public | Workflow UI + engine |
| 8080 | ai-toolbox | Loopback only (127.0.0.1) | MCP + AI gateway (`/ai`, `/mcp/*`, `/skills`, `/health`) — n8n calls `http://127.0.0.1:8080` |

## URLs

- `http://140.238.229.137/health` — Caddy health check
- `http://140.238.229.137:5678` — n8n UI
- ai-toolbox health, on the VM: `curl 127.0.0.1:8080/health`

## What lives where

| What | Where | How it gets there |
|------|-------|-------------------|
| App code (src/, skills) | Inside Docker image | Built by ai-toolbox CI, pulled via `docker-compose pull` |
| docker-compose.yml | `~/apps/server-config/` on VM | SCP'd by deploy workflow |
| Caddyfile | `~/apps/server-config/` on VM | SCP'd by deploy workflow |
| Secrets (.env) | `/etc/ai-toolbox/.env` on VM | Created manually once, never in git |
| Claude token | `/etc/ai-toolbox/.env` | `claude setup-token` on your laptop, pasted in once |
| Groww/Kite tokens | `mcp_auth` volume on VM | One-time MCP login, persisted in volume |
| n8n workflows | Docker volume on VM | Created in n8n UI, persisted in n8n_data volume |

## Logs

```bash
docker logs ai-toolbox --tail 20    # MCP + AI gateway
docker exec -it ai-toolbox claude --resume <sessionId>   # full research path of one /ai run (last 7 days)
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
