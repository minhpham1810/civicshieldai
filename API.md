# CivicShieldAI — API Documentation

Base URL: `https://civicshieldai-backend.onrender.com`

All protected endpoints require a Clerk JWT in the `Authorization` header:
```
Authorization: Bearer <clerk-jwt>
```

---

## Health

### GET `/health`

Basic health check. No authentication required.

**Response** `200 OK`
```json
{
  "status": "ok"
}
```

---

### GET `/health/db`

Database connectivity check. No authentication required.

**Response** `200 OK`
```json
{
  "status": "ok",
  "database": "connected"
}
```

**Response** `503 Service Unavailable`
```json
{
  "status": "error",
  "database": "unreachable",
  "detail": "connection refused"
}
```

---

## Webhooks

### POST `/webhooks/clerk`

Receives Clerk user lifecycle events. Verified via Svix signature headers.

**Headers:**
- `svix-id` — Webhook delivery ID
- `svix-timestamp` — Delivery timestamp
- `svix-signature` — Svix HMAC signature

**Supported Events:**

| Event | Action |
|-------|--------|
| `user.created` | Creates user record with encrypted email/name |
| `user.updated` | Updates encrypted email/name fields |

**Request Body** (Clerk webhook payload):
```json
{
  "type": "user.created",
  "data": {
    "id": "user_abc123",
    "email_addresses": [
      { "email_address": "user@example.com" }
    ],
    "first_name": "Jane",
    "last_name": "Doe"
  }
}
```

**Response** `200 OK`
```json
{
  "status": "ok",
  "event": "user.created"
}
```

**Response** `400 Bad Request` (invalid signature)
```json
{
  "detail": "Invalid webhook signature"
}
```

---

### POST `/webhooks/tally`

Receives Tally form submissions. Verified via HMAC-SHA256 signature.

**Headers:**
- `Tally-Signature` — HMAC-SHA256 hex digest

**Request Body** (Tally webhook payload):
```json
{
  "data": {
    "fields": [
      {
        "label": "Do you use MFA for email?",
        "value": "Yes",
        "key": "question_abc123"
      },
      {
        "label": "tenant_token",
        "value": "abc123_token_value",
        "key": "hidden_field_xyz"
      }
    ]
  }
}
```

The `tenant_token` hidden field links the submission to a specific tenant.

**Response** `200 OK` (success)
```json
{
  "status": "ok",
  "assessment_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Response** `200 OK` (error — always 200 to prevent Tally retries)
```json
{
  "status": "error",
  "detail": "missing tenant_token"
}
```

Possible error details: `"invalid signature"`, `"invalid json"`, `"missing tenant_token"`, `"unknown tenant"`, `"processing failed"`.

---

## Tenants

### POST `/api/tenants`

Create a new tenant with the current user as owner.

**Auth:** Clerk JWT required

**Request Body:**
```json
{
  "name": "Acme City Government",
  "slug": "acme-city-gov",
  "owner_email": "admin@acme.gov",
  "owner_full_name": "Jane Doe",
  "clerk_org_id": "org_abc123"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Organization display name |
| `slug` | string | Yes | URL-safe identifier (unique) |
| `owner_email` | string | Yes | Owner's email (will be encrypted) |
| `owner_full_name` | string | No | Owner's name (will be encrypted) |
| `clerk_org_id` | string | No | Clerk organization ID |

**Response** `201 Created`
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Acme City Government",
  "slug": "acme-city-gov",
  "is_active": true
}
```

---

### GET `/api/tenants/tally-link`

Get the tenant-specific Tally assessment form URL with the intake token pre-filled.

**Auth:** Clerk JWT required (tenant-scoped)

**Response** `200 OK`
```json
{
  "url": "https://tally.so/r/68DW4J?tenant_token=abc123_random_token",
  "token": "abc123_random_token"
}
```

**Response** `404 Not Found`
```json
{
  "detail": "No intake token configured for this tenant"
}
```

---

## Users

### GET `/api/users/me`

Get the current authenticated user's profile. Email and full name are decrypted before returning.

**Auth:** Clerk JWT required

**Response** `200 OK`
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "jane@acme.gov",
  "full_name": "Jane Doe",
  "role": "owner",
  "tenant_id": "660e8400-e29b-41d4-a716-446655440000"
}
```

**Response** `401 Unauthorized`
```json
{
  "detail": "User not found or inactive"
}
```

**Response** `403 Forbidden`
```json
{
  "detail": "User has no tenant. Complete onboarding first."
}
```

---

## Onboarding

### POST `/api/onboarding`

Idempotent auto-onboarding. Creates a tenant and seeds demo data for users who have no tenant yet. Safe to call multiple times — returns immediately if the user already has a tenant.

**Auth:** Clerk JWT required

**Request Body** (optional):
```json
{
  "org_name": "My Organization"
}
```

If `org_name` is omitted, it's derived from the user's full name (e.g., "Jane's Organization").

**Response** `200 OK` (first call — tenant created and data seeded)
```json
{
  "tenant_id": "550e8400-e29b-41d4-a716-446655440000",
  "org_name": "Jane's Organization",
  "seeded": true
}
```

**Response** `200 OK` (subsequent calls — already onboarded)
```json
{
  "tenant_id": "550e8400-e29b-41d4-a716-446655440000",
  "org_name": null,
  "seeded": false
}
```

**Seeded Demo Data:**
- 1 Tenant (with `tally_intake_token`)
- User promoted to OWNER role
- 2 Assessments: Q1 (completed, score 68.5), Q2 (in-progress)
- 8 Responses for Q1 (one per security domain)
- 1 MetricsSnapshot

---

## Dashboard

### GET `/api/dashboard`

Returns the current tenant's risk dashboard data, including the overall score, risk tier, and per-domain metric cards.

**Auth:** Clerk JWT required (tenant-scoped)

**Response** `200 OK`
```json
{
  "overall_score": 68.5,
  "risk_tier": "Medium",
  "metrics": [
    {
      "key": "mfa_coverage",
      "title": "MFA Coverage",
      "score": 45.0,
      "severity": "critical"
    },
    {
      "key": "backup_integrity",
      "title": "Backup Integrity",
      "score": 82.0,
      "severity": "low"
    },
    {
      "key": "edr_av",
      "title": "EDR / AV",
      "score": 91.0,
      "severity": "low"
    },
    {
      "key": "patch_cadence",
      "title": "Patch Cadence",
      "score": 55.0,
      "severity": "high"
    },
    {
      "key": "vulnerability_scan",
      "title": "Vulnerability Scanning",
      "score": 38.0,
      "severity": "critical"
    },
    {
      "key": "access_review",
      "title": "Privileged Access Review",
      "score": 70.0,
      "severity": "medium"
    },
    {
      "key": "incident_response",
      "title": "Incident Response Plan",
      "score": 60.0,
      "severity": "high"
    },
    {
      "key": "security_training",
      "title": "Security Training",
      "score": 77.0,
      "severity": "medium"
    }
  ],
  "total_assessments": 2,
  "completed_assessments": 1
}
```

**Severity mapping:**

| Score Range | Severity |
|------------|----------|
| < 50 | `critical` |
| 50–64 | `high` |
| 65–79 | `medium` |
| >= 80 | `low` |
| null | `none` |

**Risk tier mapping:**

| Score Range | Tier |
|------------|------|
| >= 75 | `Low` |
| 50–74 | `Medium` |
| < 50 | `High` |

**Response** `200 OK` (no data)
```json
{
  "overall_score": null,
  "risk_tier": null,
  "metrics": [],
  "total_assessments": 0,
  "completed_assessments": 0
}
```

---

## Assessments

### GET `/api/assessments`

List all assessments for the current tenant, ordered by creation date (newest first).

**Auth:** Clerk JWT required (tenant-scoped)

**Response** `200 OK`
```json
{
  "items": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "title": "Q1 2026 Cybersecurity Readiness",
      "framework": "NIST CSF",
      "status": "completed",
      "risk_score": 68.5,
      "created_at": "2026-03-09T12:00:00Z"
    },
    {
      "id": "660e8400-e29b-41d4-a716-446655440000",
      "title": "Q2 2026 Network Security Audit",
      "framework": "ISO 27001",
      "status": "in_progress",
      "risk_score": null,
      "created_at": "2026-03-09T12:00:00Z"
    }
  ],
  "total": 2
}
```

**Assessment statuses:** `draft`, `in_progress`, `completed`, `archived`

---

## Error Responses

All error responses follow this format:

```json
{
  "detail": "Human-readable error message"
}
```

| Status Code | Meaning |
|-------------|---------|
| `400` | Bad request (invalid input, webhook signature failure) |
| `401` | Unauthorized (missing/invalid JWT, user not found) |
| `403` | Forbidden (no tenant, insufficient role) |
| `404` | Not found |
| `500` | Internal server error |

---

## Authentication Details

### JWT Verification Flow

1. Client sends `Authorization: Bearer <token>` header
2. Backend fetches Clerk's JWKS (cached in memory)
3. JWT signature verified with RS256
4. `sub` claim extracted as `clerk_user_id`
5. User record looked up in database
6. `tenant_id` extracted from user record
7. All subsequent queries scoped to that tenant

### Clerk JWKS Endpoint

The backend auto-discovers the JWKS URL from the Clerk publishable key, or uses `CLERK_JWKS_URL` if explicitly set.

---

## Rate Limits

No application-level rate limiting is currently implemented. Render and Supabase enforce their own platform limits.

---

## CORS

The backend allows requests from:
- `FRONTEND_URL` (primary frontend origin)
- `FRONTEND_URLS` (comma-separated additional origins)

Methods allowed: GET, POST, PUT, DELETE, OPTIONS
Headers allowed: All
Credentials: Enabled
