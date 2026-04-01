# Vektori Tech Stack

A technical overview of the technologies powering the Vektori e-commerce analytics platform. This document is intended for engineering onboarding and internal reference.

## Overview

Vektori is a multi-tenant SaaS platform that helps e-commerce companies understand customer behavior, optimize conversion funnels, and run A/B experiments. We process roughly 50 million events per day across ~120 active customers, mostly mid-market European brands.

Everything runs on **AWS EU (Frankfurt, eu-central-1)** to ensure GDPR compliance. All customer data stays in the EU.

## Frontend

| Technology | Purpose |
|-----------|---------|
| **Next.js** | Application framework (server-side rendering, routing) |
| **React** | UI component library |
| **TypeScript** | Type safety across the entire frontend |
| **Tailwind CSS** | Utility-first styling |
| **D3.js + Recharts** | Data visualization (funnel charts, time series, cohort tables) |
| **TanStack Query** | Server state management, caching, and real-time data fetching |

The frontend is deployed on **Vercel** with edge caching for static assets. Dashboard pages use server-side rendering for initial load performance, then switch to client-side data fetching for interactivity.

### Key Frontend Features

- **Real-time dashboards**: WebSocket connections stream live event counts and conversion metrics. Updates every 5 seconds.
- **Funnel builder**: Drag-and-drop interface for defining multi-step conversion funnels. Uses D3 for the Sankey-style visualization.
- **Experiment results**: Statistical significance calculator built in-house using sequential testing methodology. Shows confidence intervals, lift estimates, and sample size progress.
- **Cohort analysis**: Matrix-style retention tables with color-coded cells. Supports behavioral and time-based cohort definitions.

## Backend

| Technology | Purpose |
|-----------|---------|
| **Python 3.12** | Primary backend language |
| **FastAPI** | API framework (async, auto-generated OpenAPI docs) |
| **Pydantic** | Request/response validation and serialization |
| **Celery** | Async task queue for heavy processing (report generation, data exports) |
| **SQLAlchemy + Alembic** | ORM and database migrations for PostgreSQL |

The backend runs on **AWS ECS (Fargate)** with auto-scaling based on CPU and request queue depth. We typically run 4-8 API containers during business hours, scaling down to 2 overnight.

### API Design

RESTful API with versioned endpoints under `/v1`. Authentication via API keys (hashed with bcrypt, stored in PostgreSQL). Rate limited to 100 requests/second per key using Redis token bucket.

Key endpoint groups:
- `/v1/events` -- Event ingestion and querying
- `/v1/funnels` -- Funnel definitions and results
- `/v1/experiments` -- A/B test management
- `/v1/cohorts` -- Cohort definitions and membership
- `/v1/export` -- Async data export jobs
- `/v1/predictions` -- ML-based churn prediction (shipped Q1 2026)

## Data Layer

### ClickHouse Cloud (Analytics)

Our analytics workhorse. ClickHouse handles all event storage and analytical queries.

- **Why ClickHouse**: Column-oriented storage is perfect for analytics queries that scan millions of rows but only touch a few columns. Queries that would take 30+ seconds in PostgreSQL complete in under a second.
- **Schema**: Events are stored in a single wide table partitioned by month and ordered by `(tenant_id, event_timestamp)`. This ordering makes tenant-scoped time-range queries extremely fast.
- **Materialized views**: Pre-aggregate daily event counts, funnel step completions, and experiment variant assignments. These power the dashboard widgets without touching raw event data.
- **Cluster**: 3-node ClickHouse Cloud cluster. ~4TB compressed data (roughly 20:1 compression ratio on raw event JSON).

### PostgreSQL 15 (Application Data)

Amazon RDS PostgreSQL handles everything that is not analytics:

- User accounts and authentication
- Project and workspace configuration
- Experiment definitions and targeting rules
- Funnel definitions
- Webhook configurations
- Billing and subscription data

### Redis 7 (Caching & Real-time)

Amazon ElastiCache Redis serves multiple purposes:

- **Session cache**: User sessions with 24-hour TTL
- **Rate limiting**: Token bucket implementation for API rate limits
- **Real-time counters**: Live event counts for dashboard widgets (using Redis HyperLogLog for unique user counts)
- **Feature flags**: LaunchDarkly-style feature flag evaluation cache
- **Pub/Sub**: WebSocket message broadcasting across API containers

## Event Processing Pipeline

This is the core of what Vektori does -- getting events from customer websites into queryable analytics.

```
SDK/API --> API Gateway --> SQS Queue --> Event Processor --> ClickHouse
                |                              |
                v                              v
            Rate Limiter               Webhook Dispatcher
             (Redis)                        (SQS)
```

1. **Ingestion**: Events arrive via the JavaScript SDK or REST API. The API gateway validates the payload, checks rate limits, and pushes to SQS.
2. **Processing**: ECS-based event processors consume from SQS in batches of 500. They enrich events (GeoIP, user agent parsing), validate schemas, and deduplicate.
3. **Storage**: Processed events are inserted into ClickHouse in batches every 5 seconds or when batch size hits 10,000, whichever comes first.
4. **Webhooks**: If the customer has webhooks configured, matching events trigger webhook deliveries via a separate SQS queue with retry logic.

End-to-end latency from SDK to dashboard visibility is typically **under 5 minutes**.

## Machine Learning

Shipped in Q1 2026 as part of the predictive analytics feature.

| Component | Technology |
|-----------|-----------|
| **Model training** | Python, scikit-learn, XGBoost |
| **Feature store** | ClickHouse materialized views |
| **Model serving** | FastAPI endpoint on dedicated ECS service |
| **Scheduling** | AWS Step Functions (daily retraining) |

The churn prediction model uses 90 days of behavioral features (session frequency, feature adoption, support ticket volume) to predict 30-day churn probability. Accuracy on holdout set: ~82% AUC.

## CI/CD

All CI/CD runs through **GitHub Actions**.

| Workflow | Trigger | Duration |
|----------|---------|----------|
| `ci.yml` | Every PR | ~6 min |
| `deploy-staging.yml` | Merge to `main` | ~4 min |
| `deploy-production.yml` | Manual + approval | ~8 min |
| `db-migrations.yml` | Manual + approval | ~2 min |

Test suite: ~1,200 unit tests (pytest), ~150 integration tests against test ClickHouse instance, plus Playwright E2E tests for critical flows (login, funnel creation, experiment setup).

## Monitoring

- **Datadog**: APM, infrastructure metrics, custom dashboards, alerting
- **Sentry**: Error tracking (frontend and backend)
- **PagerDuty**: On-call rotation and incident escalation
- **CloudWatch**: AWS-native metrics and logs as backup

Key alerts:
- P99 API latency > 2 seconds
- Error rate > 1% over 5-minute window
- Event ingestion lag > 10 minutes
- ClickHouse query queue depth > 50
- Disk usage > 80% on any node

## Local Development

```bash
# Clone repos
git clone git@github.com:vektori/platform.git
git clone git@github.com:vektori/vektori-js.git

# Backend
cd platform/backend
cp .env.example .env  # Fill in local credentials
docker-compose up -d  # PostgreSQL, Redis, ClickHouse (local)
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload  # API on localhost:8000

# Frontend
cd ../frontend
npm install
npm run dev  # Dashboard on localhost:3000
```

See the [Infrastructure README](../infrastructure/README.md) for production architecture details and the [Engineering Runbook](https://notion.so/vektori/engineering-runbook) for operational procedures.
