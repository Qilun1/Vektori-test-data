# Vektori Infrastructure

Overview of Vektori's production infrastructure and deployment processes. This document is maintained by the platform team and reflects the current state of our systems as of March 2026.

## Architecture Overview

Vektori runs a multi-tenant SaaS architecture on AWS EU (Frankfurt, eu-central-1). All customer data is processed and stored within the EU to maintain GDPR compliance.

### Core Services

| Service | Technology | Purpose |
|---------|-----------|---------|
| Web Application | Next.js (Vercel) | Customer-facing dashboard |
| API Gateway | FastAPI on AWS ECS | REST API, event ingestion |
| Analytics Engine | ClickHouse Cloud | Event storage, funnel queries, aggregations |
| Application Database | Amazon RDS PostgreSQL 15 | User accounts, project config, experiment definitions |
| Cache Layer | Amazon ElastiCache Redis 7 | Session cache, rate limiting, real-time counters |
| Message Queue | Amazon SQS | Async event processing, webhook delivery |
| Object Storage | Amazon S3 | Data exports, backups, static assets |

### Data Flow

1. Events arrive via REST API or JavaScript SDK
2. API validates and enqueues events to SQS
3. Event processor (ECS) consumes from SQS, writes to ClickHouse in batches
4. ClickHouse materialized views pre-aggregate common queries (daily counts, funnel steps)
5. Dashboard queries hit ClickHouse directly for analytics, PostgreSQL for config/metadata

### Traffic Numbers (as of Feb 2026)

- ~50M events/day ingested across all customers
- ~2,500 API requests/minute (peak hours)
- ClickHouse cluster: 3 nodes, ~4TB compressed data
- P99 dashboard query latency: 850ms
- Event ingestion to dashboard visibility: <5 minutes typical

## CI/CD Pipeline

We use **GitHub Actions** for all CI/CD workflows.

### Pipelines

- **ci.yml**: Runs on every PR. Linting, type checking, unit tests, integration tests against test ClickHouse instance.
- **deploy-staging.yml**: Auto-deploys `main` branch to staging environment.
- **deploy-production.yml**: Manual trigger with approval gate. Deploys to production ECS services with rolling update (zero-downtime).
- **db-migrations.yml**: Runs Alembic migrations against RDS. Requires manual approval for production.

### Deployment Process

1. PR merged to `main`
2. Staging auto-deploys (takes ~4 minutes)
3. Automated smoke tests run against staging
4. Team lead triggers production deploy
5. Rolling update across ECS tasks (typically 6-8 minutes)
6. Datadog monitors verify health metrics post-deploy

## Monitoring & Observability

- **Datadog**: Primary monitoring platform
  - APM traces for all API endpoints
  - Custom dashboards for event ingestion rates, query performance
  - Alerts: P99 latency > 2s, error rate > 1%, ingestion lag > 10 min
- **PagerDuty**: On-call rotation, incident escalation
- **Sentry**: Error tracking for frontend and backend
- **ClickHouse system tables**: Query performance monitoring

## Environments

| Environment | Purpose | URL |
|-------------|---------|-----|
| Production | Live customer traffic | api.vektori.io |
| Staging | Pre-production testing | staging-api.vektori.io |
| Dev | Local development | localhost:8000 |

## Security

- All inter-service communication over TLS 1.3
- VPC with private subnets for databases and internal services
- API keys hashed with bcrypt before storage
- ClickHouse access restricted to application service accounts
- Weekly automated dependency vulnerability scans via Dependabot
- SOC 2 Type II certified (audit August 2025)

## Disaster Recovery

- RDS: Automated daily backups, 30-day retention, cross-region replica in eu-west-1
- ClickHouse: Daily snapshots to S3, point-in-time recovery capability
- S3: Versioning enabled on all buckets
- RTO target: 4 hours | RPO target: 1 hour

## Runbooks

For incident response procedures, on-call rotation, and operational runbooks, see the internal Notion wiki under Engineering Runbook.

## Contact

Platform team: #platform-team on Slack
On-call: Check PagerDuty rotation schedule
