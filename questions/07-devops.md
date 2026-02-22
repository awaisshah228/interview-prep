# DevOps, Docker & CI/CD - Interview Q&A

> 15+ questions covering Docker, GitHub Actions, Terraform, monitoring, and deployment strategies

---

## Table of Contents

- [Docker](#docker)
- [CI/CD with GitHub Actions](#cicd-with-github-actions)
- [Terraform](#terraform)
- [Monitoring & Observability](#monitoring--observability)
- [Deployment Strategies](#deployment-strategies)

---

## Docker

### Q1: Docker Multi-stage Build for Node.js.

**Answer:**

```dockerfile
# Stage 1: Install dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

# Stage 2: Build application
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build
# Prune dev dependencies
RUN pnpm prune --prod

# Stage 3: Production image
FROM node:20-alpine AS runner
WORKDIR /app

# Security: run as non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nestjs
USER nestjs

# Only copy production artifacts
COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nestjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nestjs:nodejs /app/package.json ./

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/main.js"]

# Result: ~150MB vs ~1.2GB unoptimized
```

**Optimization techniques:**
```dockerfile
# 1. Use Alpine base (5MB vs 900MB)
FROM node:20-alpine  # not node:20

# 2. Layer ordering (most stable → least stable)
COPY package.json ./     # changes rarely → cached
RUN npm install          # cached if package.json didn't change
COPY . .                 # changes often → rebuilds from here

# 3. .dockerignore
node_modules
.git
*.md
.env
tests
coverage
.github

# 4. Use specific versions (reproducibility)
FROM node:20.11.0-alpine3.19  # not node:latest
```

---

### Q2: Docker Compose for local development.

**Answer:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
      - "9229:9229" # debug port
    volumes:
      - .:/app          # live reload
      - /app/node_modules # don't mount node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/trades
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    command: pnpm start:dev

  db:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: trades
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data

  localstack:
    image: localstack/localstack
    ports:
      - "4566:4566"
    environment:
      - SERVICES=sqs,sns,s3
      - DEFAULT_REGION=us-east-1

volumes:
  pgdata:
  redisdata:
```

---

### Q3: Docker networking and container communication.

**Answer:**

```
Docker Network Types:
─────────────────────
1. bridge (default)  → Containers communicate by container name
2. host              → Container shares host network
3. none              → No networking
4. overlay           → Multi-host networking (Swarm/K8s)

In Docker Compose:
- All services are on the same bridge network by default
- Services reference each other by SERVICE NAME (not IP)
- api → connects to db:5432, redis:6379 (by name)

Port mapping: HOST:CONTAINER
- "3000:3000"  → host port 3000 → container port 3000
- "5433:5432"  → host port 5433 → container port 5432

DNS resolution:
- Docker provides built-in DNS for service names
- api container: getaddrinfo('db') → returns db container's IP
```

---

## CI/CD with GitHub Actions

### Q4: Complete CI/CD pipeline for NestJS + ECS.

**Answer:**

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: trade-api
  ECS_CLUSTER: production
  ECS_SERVICE: trade-api

jobs:
  # Job 1: Lint and Test
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports: ['6379:6379']

    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v2
        with:
          version: 8

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm test --coverage
      - run: pnpm test:e2e

      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  # Job 2: Build and Push Docker Image
  build:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr

      - name: Build and push
        id: meta
        run: |
          IMAGE_TAG=${{ github.sha }}
          FULL_IMAGE=${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${IMAGE_TAG}
          docker build -t ${FULL_IMAGE} .
          docker push ${FULL_IMAGE}
          echo "tags=${FULL_IMAGE}" >> $GITHUB_OUTPUT

  # Job 3: Deploy to ECS
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production  # requires approval

    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster ${{ env.ECS_CLUSTER }} \
            --service ${{ env.ECS_SERVICE }} \
            --force-new-deployment

      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster ${{ env.ECS_CLUSTER }} \
            --services ${{ env.ECS_SERVICE }}
```

---

### Q5: GitHub Actions - Reusable workflows and matrix builds.

**Answer:**

```yaml
# Matrix build - test across multiple versions
jobs:
  test:
    strategy:
      matrix:
        node-version: [18, 20, 22]
        os: [ubuntu-latest, macos-latest]
        exclude:
          - os: macos-latest
            node-version: 18
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: pnpm test

# Reusable workflow
# .github/workflows/reusable-deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      ecs-service:
        required: true
        type: string
    secrets:
      aws-access-key:
        required: true

jobs:
  deploy:
    environment: ${{ inputs.environment }}
    steps:
      - name: Deploy
        run: |
          aws ecs update-service --service ${{ inputs.ecs-service }}

# Calling reusable workflow
jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      ecs-service: trade-api-staging
    secrets:
      aws-access-key: ${{ secrets.AWS_ACCESS_KEY_ID }}

# Caching for faster builds
- uses: actions/cache@v4
  with:
    path: ~/.pnpm-store
    key: pnpm-${{ hashFiles('pnpm-lock.yaml') }}
    restore-keys: pnpm-
```

---

## Terraform

### Q6: Terraform basics - State, modules, lifecycle.

**Answer:**

```hcl
# main.tf - ECS Fargate service
terraform {
  required_version = ">= 1.5"

  # Remote state (team collaboration)
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # state locking
    encrypt        = true
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Variables
variable "environment" {
  type    = string
  default = "production"
}

variable "app_port" {
  type    = number
  default = 3000
}

# Module usage
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "${var.environment}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = var.environment != "production"
}

# ECS Service
resource "aws_ecs_service" "api" {
  name            = "${var.environment}-trade-api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = var.environment == "production" ? 3 : 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = module.vpc.private_subnets
    security_groups = [aws_security_group.api.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "trade-api"
    container_port   = var.app_port
  }

  lifecycle {
    ignore_changes = [task_definition] # managed by CI/CD
  }
}

# Outputs
output "api_url" {
  value = "https://${aws_lb.main.dns_name}"
}
```

```bash
# Terraform workflow
terraform init        # initialize providers, backend
terraform plan        # preview changes
terraform apply       # apply changes
terraform destroy     # tear down infrastructure

# Workspaces (manage multiple environments)
terraform workspace new staging
terraform workspace select production
terraform apply -var="environment=production"
```

**Terraform vs CloudFormation:**
| Feature | Terraform | CloudFormation |
|---|---|---|
| Multi-cloud | Yes | AWS only |
| State | Managed by you | Managed by AWS |
| Language | HCL | JSON/YAML |
| Ecosystem | Rich modules | Limited |
| Drift detection | `terraform plan` | Stack drift |
| Learning curve | Moderate | Moderate |

---

## Monitoring & Observability

### Q7: Explain the three pillars of observability.

**Answer:**

```
1. LOGS (What happened)
   ├─ Structured logging (JSON format)
   ├─ Log levels: error, warn, info, debug
   ├─ Correlation IDs (trace across services)
   └─ Tools: CloudWatch Logs, ELK Stack, Datadog

2. METRICS (How is the system performing)
   ├─ Request rate, error rate, latency (RED method)
   ├─ CPU, memory, disk, network (USE method)
   ├─ Custom business metrics (trades/second, PnL)
   └─ Tools: Prometheus + Grafana, CloudWatch, New Relic

3. TRACES (Where did time go)
   ├─ Distributed tracing across microservices
   ├─ Request path visualization
   ├─ Bottleneck identification
   └─ Tools: Jaeger, AWS X-Ray, Datadog APM
```

```typescript
// Structured logging in NestJS
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest();
    const correlationId = request.headers['x-correlation-id'] || uuid();

    return next.handle().pipe(
      tap({
        next: () => {
          this.logger.log(JSON.stringify({
            correlationId,
            method: request.method,
            path: request.url,
            statusCode: 200,
            duration: Date.now() - request.startTime,
            userId: request.user?.id,
          }));
        },
        error: (error) => {
          this.logger.error(JSON.stringify({
            correlationId,
            method: request.method,
            path: request.url,
            statusCode: error.status || 500,
            error: error.message,
            stack: error.stack,
          }));
        },
      }),
    );
  }
}

// Prometheus metrics
import { Counter, Histogram } from 'prom-client';

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
});

const tradeCounter = new Counter({
  name: 'trades_total',
  help: 'Total number of trades executed',
  labelNames: ['symbol', 'type'],
});

// RED Method dashboards:
// Rate:   requests per second
// Errors: error rate (5xx/total)
// Duration: p50, p95, p99 latency
```

---

### Q8: Alerting strategy and SLOs.

**Answer:**

```
SLO (Service Level Objective):
- API latency: p99 < 200ms
- Availability: 99.9% uptime (8.76h downtime/year)
- Error rate: < 0.1%
- Trade execution: < 500ms

Alert Levels:
─────────────
🔴 CRITICAL (page on-call):
  - API down (0 successful requests for 2 min)
  - Error rate > 5% for 5 min
  - DLQ messages > 0 (trade processing failures)
  - Database connection pool exhausted
  - Certificate expiring in < 7 days

🟡 WARNING (Slack notification):
  - Latency p99 > 200ms for 10 min
  - CPU > 80% for 15 min
  - Memory > 85%
  - Disk > 80%
  - Error rate > 1% for 5 min

📊 INFO (dashboard only):
  - Deployment completed
  - Auto-scaling events
  - Background job completion

Best practices:
- Alert on symptoms (high latency) not causes (high CPU)
- Include runbook link in alert
- Avoid alert fatigue (tune thresholds)
- Use error budgets (99.9% = 43.8 min downtime/month)
```

---

## Deployment Strategies

### Q9: Compare deployment strategies.

**Answer:**

```
1. ROLLING UPDATE (default in ECS/K8s)
   [v1][v1][v1][v1]
   [v2][v1][v1][v1]  ← replace one at a time
   [v2][v2][v1][v1]
   [v2][v2][v2][v1]
   [v2][v2][v2][v2]
   ✅ Zero downtime
   ✅ Gradual rollout
   ❌ Two versions running simultaneously
   ❌ Slow rollback (roll forward or reverse)

2. BLUE/GREEN
   Blue (v1) ←── 100% traffic
   Green (v2) ← deploy and test here
   ... switch ...
   Blue (v1)
   Green (v2) ←── 100% traffic
   ✅ Instant switch
   ✅ Instant rollback (switch back)
   ❌ 2x infrastructure during deploy
   ❌ Database migrations need care

3. CANARY
   [v1 95%] [v2 5%]    ← monitor metrics
   [v1 75%] [v2 25%]   ← looks good, increase
   [v1 50%] [v2 50%]   ← still good
   [v2 100%]            ← full rollout
   ✅ Minimal blast radius
   ✅ Real traffic testing
   ❌ Complex routing
   ❌ Need good monitoring

4. RECREATE (simple)
   Stop all v1 → Start all v2
   ✅ Simple
   ✅ Clean state
   ❌ DOWNTIME
   Only use for: dev/staging, or scheduled maintenance

Your setup: Rolling update on ECS Fargate with health checks
- minimumHealthyPercent: 50%  (keep half running during deploy)
- maximumPercent: 200%        (can double capacity temporarily)
- healthCheckGracePeriod: 60s (wait before checking health)
```

---

## References & Deep Dive Resources

### Docker
| Topic | Resource |
|---|---|
| Docker Docs (Official) | [docs.docker.com](https://docs.docker.com/) |
| Dockerfile Best Practices | [Docker - Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) |
| Multi-stage Builds | [Docker - Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/) |
| Docker Compose | [Docker Compose Docs](https://docs.docker.com/compose/) |
| Docker Networking | [Docker - Networking Overview](https://docs.docker.com/network/) |
| Docker Security | [Docker - Security Best Practices](https://docs.docker.com/develop/security-best-practices/) |
| Dive (Image Inspector) | [wagoodman/dive](https://github.com/wagoodman/dive) — Explore Docker image layers |
| Node.js Docker Guide | [Node.js Docker Best Practices](https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md) |

### CI/CD
| Topic | Resource |
|---|---|
| GitHub Actions Docs | [docs.github.com/actions](https://docs.github.com/en/actions) |
| GitHub Actions Marketplace | [github.com/marketplace?type=actions](https://github.com/marketplace?type=actions) |
| Reusable Workflows | [GitHub - Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) |
| Matrix Strategy | [GitHub - Matrix Strategy](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs) |
| GitHub Actions Caching | [GitHub - Caching Dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows) |
| CI/CD Best Practices | [GitLab CI/CD Best Practices](https://docs.gitlab.com/ee/ci/pipelines/pipeline_efficiency.html) |

### Terraform
| Topic | Resource |
|---|---|
| Terraform Docs | [developer.hashicorp.com/terraform](https://developer.hashicorp.com/terraform/docs) |
| Terraform Best Practices | [Terraform - Style Guide](https://developer.hashicorp.com/terraform/language/style) |
| Terraform State | [Terraform - State Management](https://developer.hashicorp.com/terraform/language/state) |
| Terraform Modules | [Terraform - Modules](https://developer.hashicorp.com/terraform/language/modules) |
| AWS Modules | [registry.terraform.io/namespaces/terraform-aws-modules](https://registry.terraform.io/namespaces/terraform-aws-modules) |
| Terraform vs CloudFormation | [spacelift.io - Terraform vs CloudFormation](https://spacelift.io/blog/terraform-vs-cloudformation) |

### Monitoring & Observability
| Topic | Resource |
|---|---|
| Prometheus Docs | [prometheus.io/docs](https://prometheus.io/docs/) |
| Grafana Docs | [grafana.com/docs](https://grafana.com/docs/) |
| New Relic Docs | [docs.newrelic.com](https://docs.newrelic.com/) |
| OpenTelemetry | [opentelemetry.io](https://opentelemetry.io/) — Vendor-neutral observability |
| SRE Book (Free) | [sre.google/sre-book](https://sre.google/sre-book/table-of-contents/) — Google SRE Book |
| RED Method | [Grafana - RED Method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/) |
| USE Method | [Brendan Gregg - USE Method](https://www.brendangregg.com/usemethod.html) |
| SonarQube | [sonarqube.org](https://www.sonarqube.org/) |

### Deployment Strategies
| Topic | Resource |
|---|---|
| Blue/Green on AWS | [AWS - Blue/Green Deployments](https://docs.aws.amazon.com/whitepapers/latest/overview-deployment-options/bluegreen-deployments.html) |
| Canary Deployments | [martinfowler.com - Canary Release](https://martinfowler.com/bliki/CanaryRelease.html) |
| Rolling Updates (ECS) | [AWS - ECS Rolling Updates](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html) |
| Feature Flags | [launchdarkly.com/blog](https://launchdarkly.com/blog/) — Feature flag patterns |

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Prev**: [AWS Cloud](./06-aws.md) | **Next**: [Web3 & Blockchain](./08-web3-blockchain.md)
