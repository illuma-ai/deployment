# Deployment — Claude Code Project Instructions

## Project Overview

Lightweight deployment orchestrator for the Illuma AI platform. Contains NO application code — only GitHub Actions workflows and ECS task definitions.

**GitHub:** `illuma-ai/deployment` (public)

## ⚠️ DEV MUST RUN CONTINUOUSLY — auto-stop crons are DISABLED (2026-07-21)

Per an explicit standing user decision, **no dev service may auto-stop**. The two
nightly auto-stop schedules are **commented out (NOT deleted)** so the cost safety
net can be restored later by un-commenting:
- `stop-all.yml` — `schedule: cron '0 4 * * *'` (was: stop ALL dev ECS services)
- `illuma-memory-gpu.yml` — `schedule: cron '0 4 * * *'` (was: stop the g6.xlarge GPU box)

Do **NOT** re-enable these (or add any new stop schedule) unless the user explicitly
asks. Manual `workflow_dispatch` stop still works for one-off shutdowns.
**Cost note:** the illuma-memory box is a **g6.xlarge GPU** — the most expensive dev
resource — and will now run 24/7 until manually stopped.

## Structure

```
deployment/
  .github/workflows/
    deploy.yml              # Deploy individual service (manual + repository_dispatch)
    deploy-service.yml      # Reusable workflow template for service repos
    env-control.yml         # Start all services (manual + cron 8 AM EST)
    stop.yml                # Stop individual service
    stop-all.yml            # Stop all services (manual + cron 11 PM EST)
    publish-browser-plugin.yml  # Trigger browser plugin publish
  task-definitions/
    {service}/{env}.json    # ECS Fargate task definitions (11 services × 2 envs)
  docs/
    qa-environment-setup.md # QA env provisioning guide
```

## Services Managed

| Service | ECS Name | Port |
|---------|----------|------|
| chat | illuma-ai-chat-svc-{env} | 3080 |
| rag-api | illuma-ai-rag-api-svc-{env} | 8000 |
| code-executor | illuma-ai-code-executor-svc-{env} | 8088 |
| memory | illuma-ai-memory-svc-{env} | 8001 |
| m365-mcp | illuma-ai-m365-mcp-svc-{env} | 3003 |
| nexillo-mcp | illuma-ai-nexillo-mcp-svc-{env} | 3004 |
| mongodb | illuma-ai-mongodb-svc-{env} | 27017 |
| observe | illuma-ai-observe-svc-{env} | 8090 |
| clickhouse | illuma-ai-clickhouse-svc-{env} | 8123 |
| builder | illuma-ai-builder-svc-{env} | 3001 |
| agent-bridge | illuma-ai-agent-bridge-svc-{env} | 8100 |
| voice | illuma-ai-voice-svc-{env} | 8210 |

## Deployment Flow

```
Service repo push (develop/main)
  → notify-deploy.yml sends repository_dispatch
  → deploy.yml clones service repo via PRIVATE_REPO_PAT
  → Builds Docker image → pushes to ECR (illuma-ai-{service}-{env})
  → Renders task def → deploys to ECS
  → Configures auto-scaling (min=1, max=2, CPU 70%)
```

## Key Rules

- **ALL resources use `illuma-ai-*` prefix** — NEVER gaavi/codevakure/librechat
- **Domains:** `dev-illuma.gaavi.ai` (dev), `illuma.gaavi.ai` (prod) — chat
- **Domains:** `observe-dev.gaavi.ai` (dev), `observe.gaavi.ai` (prod) — observe
- **Domains:** `builder-dev.gaavi.ai` (dev), `builder.gaavi.ai` (prod) — builder
- **Domains:** `bridge-dev.gaavi.ai` (dev), `bridge.gaavi.ai` (prod) — agent-bridge
- **Domains:** `voice-dev.gaavi.ai` (dev), `voice.gaavi.ai` (prod) — voice
  (service name `voice` → repo `illuma-ai/illuma-voice`; mapping in deploy.yml
  "Clone service repo" step)
  (one subdomain + one CloudFront distribution PER service, all origin to the
  shared ALB `illuma-ai-alb-{env}` via a host-header listener rule — never
  path/context-path routing on a shared domain; DNS lives at GoDaddy, not
  Route53)
- **Branch mapping:** `develop` → dev, `main` → prod
- **AWS Account:** 423159469683, Region: us-east-1
- Task definitions reference secrets from `illuma-ai/{env}/{service}` in AWS Secrets Manager
- Container names match `illuma-ai-{service}`

## Adding a New Service

**Repo-side (this repo):**
1. Create `task-definitions/{service}/dev.json` and `prod.json` (copy the closest
   existing service as a template — `deploy.yml` derives everything else, incl.
   image repo name, container name, and log group, purely from `{service}` +
   `{env}`; NO per-service branching needed there unless the GitHub repo name
   differs from the service name, e.g. `observe` → `illuma-ai/observability`
   — only then add a case to the "Clone service repo" step)
2. Add `{service}` to the `service` choice list in `deploy.yml` AND `stop.yml`
3. Add `{service}` to the `SERVICES=`/`ECS_SERVICES=` string in `env-control.yml`
   and `stop-all.yml`
4. Add the service to the table above + a **Domains** line below (this repo owns
   NO network resources — see the AWS/DNS steps below)

**AWS-side (one-time, per environment — console or CLI, NOT scripted here):**
5. Create ECR repo: `illuma-ai-{service}-{env}`
6. Create Secrets Manager entry: `illuma-ai/{env}/{service}` (JSON blob, one key
   per env var the task definition's `secrets[]` array references — the ARN
   suffix AWS assigns on creation must be substituted into the task-def JSON)
7. Create CloudWatch log group: `/ecs/illuma-ai-{service}-{env}`
8. Create an ALB target group `illuma-ai-{service}-tg-{env}` (health check path
   = the service's `/health`) + register it as a listener rule on
   `illuma-ai-alb-{env}` matched by `host-header = {service}[-dev].gaavi.ai`
   (see the existing `observe`/`builder` rules on `illuma-ai-alb-prod` listener
   :80 for the exact pattern — **host-header routing, one rule per service**,
   never path-based)
9. Create an ECS service `illuma-ai-{service}-svc-{env}` on cluster
   `illuma-ai-ecs-cluster-{env}` (Fargate, `awsvpc`, attached to the target
   group from step 8) — auto-scaling min=1/max=2/CPU 70% per the deploy flow
10. Create a CloudFront distribution (origin = the ALB, `http-only`,
    `redirect-to-https`) with alias `{service}[-dev].gaavi.ai` + an ACM cert for
    that alias (us-east-1, CloudFront requires the cert in us-east-1)
11. Add the DNS record at **GoDaddy** (the domain registrar — NOT Route53, this
    AWS account has no hosted zone for `gaavi.ai`): CNAME
    `{service}[-dev].gaavi.ai` → the new distribution's `*.cloudfront.net` domain
