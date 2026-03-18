# Illuma AI — Deployment Guide

## Workflows

This repo has four workflows. All are triggered from the **Actions** tab.

---

### 1. Deploy Service

Builds and deploys a single service.

| Input | Options | Description |
|-------|---------|-------------|
| **service** | chat, rag-api, code-executor, memory, m365-mcp | Which service |
| **environment** | dev, prod | Target environment |
| **ref** | Any branch or tag | Code version to deploy |
| **cpu_memory** | default, 1 vCPU / 2 GB, 2 vCPU / 4 GB, 2 vCPU / 8 GB, 4 vCPU / 8 GB, 8 vCPU / 16 GB | Resource allocation |
| **min_tasks** | 1, 2, 3 | Auto-scaling minimum |
| **max_tasks** | 1, 2, 3 | Auto-scaling maximum |

Also runs automatically when code is pushed to `develop` (→ dev) or `main` (→ prod).

---

### 2. Stop Service

Stops a single service (scales to 0).

| Input | Options | Description |
|-------|---------|-------------|
| **service** | chat, rag-api, code-executor, memory, m365-mcp | Which service |
| **environment** | dev, prod | Target environment |

---

### 3. Start All Services

Starts all services at once. Use this for demos.

| Input | Options | Description |
|-------|---------|-------------|
| **environment** | dev, prod | Target environment |
| **cpu_memory** | default, 1 vCPU / 2 GB, 2 vCPU / 4 GB, 2 vCPU / 8 GB, 4 vCPU / 8 GB, 8 vCPU / 16 GB | Resource override for all services |
| **min_tasks** | 1, 2, 3 | Tasks per service |

Runs automatically at **8 AM EST** every day (dev).

---

### 4. Stop All Services

Stops all services at once (scales everything to 0).

| Input | Options | Description |
|-------|---------|-------------|
| **environment** | dev, prod | Target environment |

Runs automatically at **11 PM EST** every day (dev).

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
✓ Configure auto-scaling  — sets min/max task scaling
✓ Deployment summary      — final status with resource details
```

Click on any failed step to see full logs.

---

## Common Tasks

**Deploy a feature branch for testing:**
Deploy Service → service = yours, environment = dev, ref = your-branch-name

**Start everything for a demo:**
Start All Services → environment = dev

**Shut down after demo:**
Stop All Services → environment = dev

**Scale up for load testing:**
Deploy Service → cpu_memory = 4 vCPU / 8 GB, min_tasks = 2, max_tasks = 3

**Rollback to a previous version:**
Deploy Service → ref = previous branch or tag

---

## Troubleshooting

**Deploy failed at "Build & push image"**
Check the Docker build logs — usually a dependency or Dockerfile issue.

**Deploy failed at "Deploy"**
Container is crashing. Check service logs for missing environment variables.

**"Failed to contact the origin"**
Services are stopped. Run Start All Services.
