# Vektori REST API Reference

**Base URL:** `https://api.vektori.io/v1`
**Last updated:** September 2025

## Authentication

All API requests require an API key passed in the `Authorization` header:

```
Authorization: Bearer vk_live_your_api_key_here
```

API keys can be generated in the Vektori dashboard under Settings > API Keys. Each key is scoped to a single project.

## Rate Limits

- **100 requests/second** per API key
- Rate limit headers are included in every response:
  - `X-RateLimit-Limit`: requests allowed per window
  - `X-RateLimit-Remaining`: requests remaining
  - `X-RateLimit-Reset`: Unix timestamp when window resets
- Exceeding limits returns `429 Too Many Requests`

## Endpoints

### Events

#### POST /events

Ingest a single event or batch of events.

```json
{
  "events": [
    {
      "event": "product_viewed",
      "userId": "user-123",
      "timestamp": "2025-09-15T10:30:00Z",
      "properties": {
        "productId": "SKU-100",
        "price": 29.99,
        "currency": "EUR"
      }
    }
  ]
}
```

**Response:** `202 Accepted`

Events are processed asynchronously. They typically appear in dashboards within 5 minutes of ingestion.

#### GET /events

Query raw events with filters.

| Parameter | Type | Description |
|-----------|------|-------------|
| `userId` | string | Filter by user ID |
| `event` | string | Filter by event name |
| `from` | ISO 8601 | Start date |
| `to` | ISO 8601 | End date |
| `limit` | integer | Max results (default 100, max 1000) |

### Funnels

#### POST /funnels

Create a funnel definition.

```json
{
  "name": "Checkout Flow",
  "steps": [
    {"event": "product_viewed"},
    {"event": "add_to_cart"},
    {"event": "checkout_started"},
    {"event": "purchase_completed"}
  ],
  "window": "7d"
}
```

#### GET /funnels/{funnel_id}/results

Returns funnel conversion data including step-by-step drop-off rates, median time between steps, and conversion rate.

### Experiments (A/B Testing)

#### POST /experiments

Create a new A/B test experiment.

```json
{
  "name": "Checkout Button Color",
  "variants": [
    {"name": "control", "weight": 50},
    {"name": "green_button", "weight": 50}
  ],
  "targetMetric": "purchase_completed",
  "minimumSampleSize": 1000
}
```

#### GET /experiments/{experiment_id}

Returns experiment status, variant performance, statistical significance (using sequential testing), and recommendation.

#### PATCH /experiments/{experiment_id}

Update experiment (pause, resume, or conclude).

### Cohorts

#### POST /cohorts

Define a user cohort based on behavioral criteria.

```json
{
  "name": "Power Users Q1",
  "criteria": {
    "event": "purchase_completed",
    "count": {"gte": 3},
    "timeframe": "90d"
  }
}
```

#### GET /cohorts/{cohort_id}/users

Returns list of user IDs matching the cohort definition.

### Export

#### POST /export

Request a data export. Supported formats: CSV, JSON, Parquet.

```json
{
  "type": "events",
  "format": "parquet",
  "dateRange": {
    "from": "2025-01-01",
    "to": "2025-09-30"
  },
  "filters": {
    "events": ["purchase_completed", "add_to_cart"]
  }
}
```

**Response:** Returns a `jobId`. Poll `GET /export/{jobId}` for status and download URL.

Export files are available for 72 hours after generation.

## Webhooks

Configure webhooks in the dashboard to receive real-time notifications when specific events occur.

**Webhook payload (legacy format):**

```json
{
  "type": "event",
  "data": {
    "event": "purchase_completed",
    "userId": "user-123",
    "timestamp": "2025-09-15T10:30:00Z"
  },
  "signature": "sha256=..."
}
```

Webhook requests include an `X-Vektori-Signature` header for payload verification. Webhooks that return non-2xx responses will be retried up to 3 times with exponential backoff.

## Error Codes

| Code | Description |
|------|-------------|
| 400 | Invalid request body or parameters |
| 401 | Missing or invalid API key |
| 403 | Insufficient permissions |
| 404 | Resource not found |
| 429 | Rate limit exceeded |
| 500 | Internal server error |

## SDKs

Official SDKs are available for JavaScript (see [vektori-js](../sdk/README.md)). Python and Ruby SDKs are on the roadmap.
