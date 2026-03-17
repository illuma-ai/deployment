# Illuma AI — Deployment Guide

## Workflows

This repo has two workflows. Both are triggered from the **Actions** tab.

---

### 1. Deploy Service

Deploys, stops, or starts a single service.

**How to run:**
1. Go to **Actions → Deploy Service**
2. Click **"Run workflow"**
3. Fill in the inputs and click **"Run workflow"**

| Input | Options | Description |
|-------|---------|-------------|
| **service** | chat, rag-api, code-executor, memory, m365-mcp | Which service |
| **environment** | dev, prod | Target environment |
| **action** | deploy, stop, start | What to do |
| **ref** | Any branch or tag | Code version to deploy (only used with `deploy` action) |

#### What each action does

- **deploy** — Builds a new Docker image from the selected branch, pushes it, and deploys it. Waits for health check to pass before marking complete.
- **stop** — Scales the service to 0 running tasks. Use this to shut down a single service.
- **start** — Scales the service back up to its default task count.

#### Auto-deploy

This workflow also runs automatically when code is pushed to `develop` or `main` on any service repo. No manual trigger needed.

| Branch pushed | Deploys to |
|---------------|-----------|
| `develop` | dev |
| `main` | prod |

---

### 2. Environment Control

Starts or stops **all services at once**. Use this for demos.

**How to run:**
1. Go to **Actions → Environment Control**
2. Click **"Run workflow"**
3. Fill in the inputs and click **"Run workflow"**

| Input | Options | Description |
|-------|---------|-------------|
| **action** | start-all, stop-all | Start or stop everything |
| **environment** | dev, prod | Target environment |

#### Scheduled automation

| Schedule | Action | Days |
|----------|--------|------|
| **8 AM EST** | auto start-all (dev) | Monday – Friday |
| **11 PM EST** | auto stop-all (dev) | Every night |

Services start automatically on weekday mornings and stop every night to save costs.

---

## Monitoring a Deployment

Click on any workflow run to see real-time progress:

```
✓ Resolve Inputs          — identifies service, environment, branch
✓ Fetch source code       — pulls latest code
✓ Configure cloud         — authenticates
✓ Build & push image      — builds Docker image, pushes to registry
✓ Update task definition  — updates service config with new image
✓ Deploy                  — deploys and waits for health check
✓ Deployment summary      — final status
```

Click on any failed step to see full logs.

---

## Common Tasks

**Deploy a feature branch for testing:**
Deploy Service → service = yours, environment = dev, action = deploy, ref = your-branch-name

**Start everything for a demo:**
Environment Control → action = start-all, environment = dev

**Shut down after demo:**
Environment Control → action = stop-all, environment = dev

**Restart one service:**
Deploy Service → action = stop, then action = start

**Rollback to a previous version:**
Deploy Service → action = deploy, ref = previous branch or tag

---

## Troubleshooting

**Deploy failed at "Build & push image"**
Check the Docker build logs — usually a dependency or Dockerfile issue.

**Deploy failed at "Deploy"**
Container is crashing. Check service logs for missing environment variables.

**"Failed to contact the origin" on dev-illuma.gaavi.ai**
Services are stopped. Run Environment Control → start-all.
