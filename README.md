# Illuma AI — Deployment Orchestrator

Central CI/CD pipeline for all Illuma AI platform services. This public repo runs all deployment workflows with **unlimited free GitHub Actions minutes**.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  illuma-ai/deployment (this repo)         │
│                                                           │
│  Workflows:                    Task Definitions:          │
│  ├── deploy.yml                ├── chat/dev.json          │
│  └── env-control.yml           ├── rag-api/dev.json       │
│                                ├── code-executor/dev.json │
│                                ├── memory/dev.json        │
│                                └── m365-mcp/dev.json      │
└─────────────────┬─────────────────────────────────────────┘
                  │
    ┌─────────────┼─────────────────────────────┐
    │   Clones private repos at build time       │
    │   via PRIVATE_REPO_PAT secret              │
    ▼             ▼              ▼               ▼
┌────────┐ ┌──────────┐ ┌───────────────┐ ┌─────────┐ ┌──────────┐
│  chat  │ │  rag-api │ │ code-executor │ │ memory  │ │ m365-mcp │
│(private)│ │(private) │ │  (private)    │ │(private)│ │(private) │
└────────┘ └──────────┘ └───────────────┘ └─────────┘ └──────────┘
```

## Services

| Service | Port | CPU/Mem | Min Tasks | Health Check |
|---------|------|---------|-----------|--------------|
| chat | 3080 | 8 vCPU / 16 GB | 2 | `GET /api/health` |
| rag-api | 8000 | 8 vCPU / 16 GB | 2 | `GET /health` |
| code-executor | 8088 | 2 vCPU / 4 GB | 2 | `GET /health` |
| memory | 8001 | 1 vCPU / 2 GB | 2 | `GET /health` |
| m365-mcp | 3003 | 0.25 vCPU / 512 MB | 1 | `GET /health` |

## Workflows

### Deploy Service (`deploy.yml`)

Deploy, stop, or start individual services via the GitHub Actions UI.

**Inputs:**
- **service**: chat, rag-api, code-executor, memory, m365-mcp
- **environment**: dev, prod
- **action**: deploy (build + push + ECS update), stop (scale to 0), start (scale to min tasks)
- **ref**: Git branch/tag to deploy (default: develop)

### Environment Control (`env-control.yml`)

Start or stop ALL services at once. Includes a nightly auto-stop cron for dev.

**Inputs:**
- **action**: start-all, stop-all
- **environment**: dev, prod

**Schedule:** Auto-stops dev at 1 AM UTC (8 PM EST) nightly.

## Auto-Deploy (via repository_dispatch)

Each private repo can auto-trigger deployments by adding a small workflow:

```yaml
# .github/workflows/notify-deploy.yml
name: Trigger Deployment
on:
  push:
    branches: [develop, main]
jobs:
  dispatch:
    runs-on: ubuntu-latest
    steps:
      - uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.DEPLOYMENT_PAT }}
          repository: illuma-ai/deployment
          event-type: deploy-{SERVICE_NAME}
          client-payload: '{"ref":"${{ github.ref }}","sha":"${{ github.sha }}","branch":"${{ github.ref_name }}"}'
```

## Naming Conventions

### AWS Resources: `illuma-ai-{service}-{resource}-{env}`

| Resource | Pattern | Example |
|----------|---------|---------|
| ECS Cluster | `illuma-ai-ecs-cluster-{env}` | `illuma-ai-ecs-cluster-dev` |
| ECS Service | `illuma-ai-{svc}-svc-{env}` | `illuma-ai-chat-svc-dev` |
| Task Definition | `illuma-ai-{svc}-td-{env}` | `illuma-ai-chat-td-dev` |
| ECR Repository | `illuma-ai-{svc}-{env}` | `illuma-ai-chat-dev` |
| CloudWatch Logs | `/ecs/illuma-ai-{svc}-{env}` | `/ecs/illuma-ai-chat-dev` |
| Secrets Manager | `illuma-ai/{env}/{svc}` | `illuma-ai/dev/chat` |
| ALB | `illuma-ai-alb-{env}` | `illuma-ai-alb-dev` |
| Target Group | `illuma-ai-{svc}-tg-{env}` | `illuma-ai-chat-tg-dev` |

## Secrets Required

### On this repo (illuma-ai/deployment)

| Secret | Purpose |
|--------|---------|
| `PRIVATE_REPO_PAT` | GitHub PAT to clone private repos |
| `AWS_ACCESS_KEY_ID_DEV` | AWS credentials (dev) |
| `AWS_SECRET_ACCESS_KEY_DEV` | AWS credentials (dev) |
| `AWS_ACCESS_KEY_ID_PROD` | AWS credentials (prod) |
| `AWS_SECRET_ACCESS_KEY_PROD` | AWS credentials (prod) |

### On each private repo (for auto-deploy)

| Secret | Purpose |
|--------|---------|
| `DEPLOYMENT_PAT` | GitHub PAT to trigger repository_dispatch |

## Security

- This repo is **public** but contains NO secrets or source code
- Task definitions contain only ARN references (not actual secret values)
- Only collaborators with **write access** can trigger `workflow_dispatch`
- Production environment requires reviewer approval
- Source code is cloned at runtime via encrypted PAT

## Cost Optimization

Use `env-control.yml` to stop all services when not in use:

| Mode | Monthly ECS Cost |
|------|-----------------|
| Always-on (5 services, min tasks) | ~$800 |
| Demo-only (4h/day) | ~$130 |

Databases (MongoDB, PostgreSQL, Redis) and ALB stay running — they're cheap and enable instant restart.
