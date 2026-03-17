# Illuma AI — Deployment Guide

Central CI/CD pipeline for all Illuma AI platform services.

## Quick Reference

| Service | Dev URL | Health Check |
|---------|---------|-------------|
| chat | https://dev-illuma.gaavi.ai | `GET /api/health` |
| rag-api | (internal) :8000 | `GET /health` |
| code-executor | (internal) :8088 | `GET /health` |
| memory | (internal) :8001 | `GET /health` |
| m365-mcp | (internal) :3003 | `GET /health` |

---

## How to Deploy

### Option 1: Automatic (recommended)

Push code to `develop` or `main` branch on any service repo. Deployment triggers automatically.

```
git push origin develop    →  deploys to DEV
git push origin main       →  deploys to PROD
```

That's it. The pipeline will:
1. Build the Docker image
2. Push to Amazon ECR
3. Update the ECS service
4. Wait for health check to pass
5. Show deployment summary

### Option 2: Manual (via GitHub Actions UI)

1. Go to [**Actions → Deploy Service**](https://github.com/illuma-ai/deployment/actions/workflows/deploy.yml)
2. Click **"Run workflow"**
3. Select:
   - **service** — which service to deploy (chat, rag-api, code-executor, memory, m365-mcp)
   - **environment** — dev or prod
   - **action** — deploy, stop, or start
   - **ref** — branch or tag to deploy (default: main)
4. Click **"Run workflow"**

#### Actions explained

| Action | What it does |
|--------|-------------|
| **deploy** | Builds new image, pushes to ECR, updates ECS service, waits for healthy |
| **stop** | Scales service to 0 tasks (saves cost, no downtime wait) |
| **start** | Scales service back to minimum tasks |

---

## How to Start/Stop All Services

Use this for **demo mode** — spin everything up before a demo, shut down after.

1. Go to [**Actions → Environment Control**](https://github.com/illuma-ai/deployment/actions/workflows/env-control.yml)
2. Click **"Run workflow"**
3. Select:
   - **action** — `start-all` or `stop-all`
   - **environment** — dev or prod
4. Click **"Run workflow"**

### Auto-stop

Dev environment **automatically stops every night at 8 PM EST** (1 AM UTC) to save costs. Run `start-all` the next morning when needed.

### Cost impact

| Mode | Monthly ECS Cost |
|------|-----------------|
| Always-on (all services running 24/7) | ~$800 |
| Demo-only (start before demo, stop after) | ~$130 |

> **Tip**: Databases (MongoDB, Redis) and the load balancer stay running even when services are stopped. This means `start-all` brings services back in ~2 minutes — no data loss.

---

## Monitoring a Deployment

Every deployment run shows real-time progress. Click on any run to see:

```
Step 1: ✓ Resolve Inputs          — identifies service, environment, branch
Step 2: ✓ Clone service repo      — pulls latest code from private repo
Step 3: ✓ Configure AWS           — authenticates with AWS
Step 4: ✓ Login to ECR            — authenticates with container registry
Step 5: ✓ Build & push image      — builds Docker image, pushes to ECR
Step 6: ✓ Render task definition  — updates ECS task with new image
Step 7: ✓ Deploy to ECS           — deploys to ECS, waits for healthy
Step 8: ✓ Deployment summary      — shows final status
```

If any step fails, click on it to see the full logs.

### Deployment Summary

After each successful deploy, a summary is posted showing:
- Service name
- Environment
- Branch deployed
- Docker image tag

---

## Service Repos

| Service | Repo | What it does |
|---------|------|-------------|
| chat | [illuma-ai/chat](https://github.com/illuma-ai/chat) | Main chat application |
| rag-api | [illuma-ai/rag-api](https://github.com/illuma-ai/rag-api) | RAG document processing & search |
| code-executor | [illuma-ai/code-executor](https://github.com/illuma-ai/code-executor) | Secure code execution |
| memory | [illuma-ai/memory](https://github.com/illuma-ai/memory) | Agent memory system |
| m365-mcp | [illuma-ai/m365-mcp](https://github.com/illuma-ai/m365-mcp) | Microsoft 365 connector |

### Branch → Environment mapping

| Branch | Environment | Auto-deploy? |
|--------|-------------|-------------|
| `develop` | dev | Yes |
| `main` | prod | Yes |
| Any other | — | No (use manual deploy with `ref` input) |

---

## Common Tasks

### "I want to deploy my branch to dev for testing"

1. Go to **Actions → Deploy Service → Run workflow**
2. Set **service** = your service, **environment** = dev, **action** = deploy, **ref** = your-branch-name
3. Run it

### "Services are down / I need to start everything"

1. Go to **Actions → Environment Control → Run workflow**
2. Set **action** = start-all, **environment** = dev
3. Run it — all services will be up in ~2 minutes

### "Demo is over, shut everything down"

1. Go to **Actions → Environment Control → Run workflow**
2. Set **action** = stop-all, **environment** = dev
3. Run it — (or just wait, dev auto-stops at 8 PM EST)

### "I want to restart just one service"

1. Go to **Actions → Deploy Service → Run workflow**
2. Set **action** = stop, run it
3. Then set **action** = start, run it

### "Something is broken, I need to rollback"

1. Go to **Actions → Deploy Service → Run workflow**
2. Set **ref** to the previous known-good branch or tag
3. Set **action** = deploy
4. Run it — the previous version will be built and deployed

---

## Architecture Overview

```
Developer pushes to develop/main
        │
        ▼
Private repo (illuma-ai/chat, etc.)
        │ notify-deploy.yml triggers
        ▼
illuma-ai/deployment (this repo)
        │ deploy.yml runs
        ▼
┌─────────────────────────────┐
│ 1. Clone private repo       │
│ 2. Build Docker image       │
│ 3. Push to Amazon ECR       │
│ 4. Deploy to Amazon ECS     │
│ 5. Wait for health check    │
└─────────────────────────────┘
        │
        ▼
CloudFront → ALB → ECS Tasks
https://dev-illuma.gaavi.ai
```

### Resource Allocation

| Service | CPU | Memory | Min Tasks |
|---------|-----|--------|-----------|
| chat | 8 vCPU | 16 GB | 2 |
| rag-api | 8 vCPU | 16 GB | 2 |
| code-executor | 2 vCPU | 4 GB | 2 |
| memory | 1 vCPU | 2 GB | 2 |
| m365-mcp | 0.25 vCPU | 512 MB | 1 |

---

## Secrets & Permissions

Only users with **write access** to this repo can trigger workflows.

### Required secrets (already configured)

| Secret | Where | Purpose |
|--------|-------|---------|
| `PRIVATE_REPO_PAT` | This repo | Clone private service repos at build time |
| `AWS_ACCESS_KEY_ID_DEV` | This repo | AWS credentials for dev deploys |
| `AWS_SECRET_ACCESS_KEY_DEV` | This repo | AWS credentials for dev deploys |
| `DEPLOYMENT_PAT` | Each private repo | Trigger auto-deploy on push |

---

## Troubleshooting

### Deploy failed at "Build & push image"
- Check the Docker build logs — likely a dependency or Dockerfile issue
- Try building locally first: `docker build .` in the service repo

### Deploy failed at "Deploy to ECS"
- The health check is failing — the container starts but crashes
- Check CloudWatch logs: `/ecs/illuma-ai-{service}-dev`
- Common cause: missing environment variables or secrets

### Services show 0 running tasks
- Services are probably stopped (either manually or by nightly auto-stop)
- Run **Environment Control → start-all**

### "Failed to contact the origin" on dev-illuma.gaavi.ai
- Services are stopped — run start-all
- Or CloudFront/ALB issue — check if ALB returns 200 directly: `curl http://illuma-ai-alb-dev-1731619641.us-east-1.elb.amazonaws.com/api/health`
