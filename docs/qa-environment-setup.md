# QA Environment Setup Guide

## Status: NOT YET PROVISIONED

The QA environment infrastructure does not exist yet. This document describes what needs to be created to bring up a QA environment.

---

## What Needs to Be Done

### 1. AWS Infrastructure (ECS)

Create the following AWS resources for the `qa` environment:

#### ECS Cluster
```bash
aws ecs create-cluster --cluster-name illuma-ai-ecs-cluster-qa
```

#### ECR Repositories (one per service)
```bash
for svc in chat rag-api code-executor memory m365-mcp; do
  aws ecr create-repository --repository-name "illuma-ai-${svc}-qa"
done
```

#### IAM Roles
Create copies of the dev roles for qa:
- `illuma-ai-ecs-execution-role-qa` — needs permissions for:
  - ECR pull (`ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`)
  - CloudWatch Logs (`logs:CreateLogStream`, `logs:PutLogEvents`)
  - Secrets Manager (`secretsmanager:GetSecretValue`) for `illuma-ai/qa/*`
- `illuma-ai-ecs-task-role-qa` — needs permissions for:
  - S3 access (for code-executor file storage)
  - Any other service-specific permissions

#### CloudWatch Log Groups
```bash
for svc in chat rag-api code-executor memory m365-mcp; do
  aws logs create-log-group --log-group-name "/ecs/illuma-ai-${svc}-qa"
done
```

#### VPC & Networking
Either:
- **Option A (recommended)**: Reuse the dev VPC with separate subnets for QA
- **Option B**: Create a new VPC `illuma-ai-vpc-qa` with its own subnets

Need:
- At least 2 subnets in different AZs (for ALB)
- Security groups allowing inter-service communication
- NAT gateway for internet access (ECR pulls, external APIs)

#### ALB + Target Groups
```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name illuma-ai-alb-qa \
  --subnets subnet-xxx subnet-yyy \
  --security-groups sg-xxx

# Create target groups (one per service)
for svc in chat rag-api code-executor memory m365-mcp; do
  PORT=$(case $svc in chat) echo 3080;; rag-api) echo 8000;; code-executor) echo 8088;; memory) echo 8001;; m365-mcp) echo 3003;; esac)
  aws elbv2 create-target-group \
    --name "illuma-ai-${svc}-tg-qa" \
    --protocol HTTP \
    --port $PORT \
    --vpc-id vpc-xxx \
    --target-type ip \
    --health-check-path "/health"
done
```

#### ECS Services
```bash
for svc in chat rag-api code-executor memory m365-mcp; do
  TASKS=$(case $svc in chat|rag-api|code-executor|memory) echo 2;; m365-mcp) echo 1;; esac)
  aws ecs create-service \
    --cluster illuma-ai-ecs-cluster-qa \
    --service-name "illuma-ai-${svc}-svc-qa" \
    --task-definition "illuma-ai-${svc}-td-qa" \
    --desired-count $TASKS \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=DISABLED}" \
    --load-balancers "targetGroupArn=arn:...,containerName=illuma-ai-${svc},containerPort=$PORT"
done
```

### 2. Secrets Manager

Create secrets for each service under the `illuma-ai/qa/` path:

```bash
for svc in chat rag-api code-executor memory m365-mcp; do
  aws secretsmanager create-secret \
    --name "illuma-ai/qa/${svc}" \
    --description "QA environment secrets for ${svc}" \
    --secret-string '{"APP_SECRET":"placeholder"}'
done
```

**Required secrets per service:**

#### chat (`illuma-ai/qa/chat`)
- MONGO_URI, JWT_SECRET, JWT_REFRESH_SECRET, CREDS_KEY, CREDS_IV
- DOMAIN_CLIENT, DOMAIN_SERVER, HOST, PORT
- RAG_API_URL, CODE_EXECUTOR_BASEURL, SIMPLEMEM_API_URL
- BEDROCK_AWS_ACCESS_KEY_ID, BEDROCK_AWS_SECRET_ACCESS_KEY, BEDROCK_AWS_DEFAULT_REGION
- AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION, AWS_BUCKET_NAME
- All LLM provider keys (SERPER_API_KEY, FIRECRAWL_API_KEY, etc.)

#### rag-api (`illuma-ai/qa/rag-api`)
- DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME (PostgreSQL)
- JWT_SECRET
- AWS credentials for Textract/Bedrock OCR
- Embedding model config

#### code-executor (`illuma-ai/qa/code-executor`)
- CODE_API_KEY
- AWS_BUCKET_NAME, AWS_REGION, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
- FILE_STRATEGY=s3
- USE_REDIS, REDIS_URI

#### memory (`illuma-ai/qa/memory`)
- PostgreSQL connection (DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME)
- AWS Bedrock credentials (for embeddings)

#### m365-mcp (`illuma-ai/qa/m365-mcp`)
- TENANT_ID, CLIENT_ID, CLIENT_SECRET
- MS365_MCP_OUTPUT_FORMAT

### 3. Databases

#### MongoDB (for chat)
- Create a separate QA database/cluster in MongoDB Atlas
- Or use same cluster with a different database name: `illuma-qa`

#### PostgreSQL (for rag-api and memory)
- Create a QA database in existing RDS instance: `CREATE DATABASE illuma_qa;`
- Enable pgvector extension: `CREATE EXTENSION vector;`
- Or create a separate RDS instance for isolation

#### Redis (for chat + code-executor)
- Create a separate ElastiCache Redis cluster for QA
- Or use same cluster with key prefix isolation: `qa:*`

### 4. DNS

Set up DNS for QA:
- `qa-illuma.gaavi.ai` → ALB DNS (or use a different domain)

### 5. Task Definitions

Create `qa.json` in each service's task-definitions directory:
```
task-definitions/
├── chat/
│   ├── dev.json
│   ├── qa.json      ← NEW
│   └── prod.json
├── rag-api/
│   ├── dev.json
│   ├── qa.json      ← NEW
│   └── prod.json
...
```

### 6. Update Workflows

Add `qa` to the environment dropdown in `deploy.yml` and `env-control.yml`:
```yaml
environment:
  type: choice
  options:
    - dev
    - qa      # ← Add this
    - prod
```

### 7. GitHub Secrets

Add QA-specific AWS credentials:
- `AWS_ACCESS_KEY_ID_QA`
- `AWS_SECRET_ACCESS_KEY_QA`

Update credential selection in workflows:
```yaml
aws-access-key-id: ${{ inputs.environment == 'prod' && secrets.AWS_ACCESS_KEY_ID_PROD || inputs.environment == 'qa' && secrets.AWS_ACCESS_KEY_ID_QA || secrets.AWS_ACCESS_KEY_ID_DEV }}
```

---

## Cost Estimate (QA Environment)

| Resource | Monthly Cost |
|----------|-------------|
| ECS Tasks (all services, min tasks) | ~$800 |
| ECS Tasks (4h/day demo mode) | ~$130 |
| RDS PostgreSQL (db.t3.micro) | ~$15 |
| ElastiCache Redis (cache.t3.micro) | ~$13 |
| ALB | ~$22 |
| **Total (always-on)** | **~$850** |
| **Total (demo mode)** | **~$180** |

**Recommendation**: Use demo mode (env-control.yml) for QA — start before testing, stop after.

---

## Checklist

- [ ] Create ECS cluster `illuma-ai-ecs-cluster-qa`
- [ ] Create ECR repos for all 5 services
- [ ] Create IAM roles (execution + task)
- [ ] Create CloudWatch log groups
- [ ] Set up VPC networking (subnets, security groups, NAT)
- [ ] Create ALB + target groups
- [ ] Create ECS services
- [ ] Create Secrets Manager entries with real values
- [ ] Set up databases (MongoDB, PostgreSQL, Redis)
- [ ] Configure DNS
- [ ] Create qa.json task definitions
- [ ] Add qa to workflow dropdowns
- [ ] Add QA AWS credentials to GitHub secrets
- [ ] Test deploy pipeline for QA
- [ ] Test env-control start/stop for QA
