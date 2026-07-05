# Backend API Specification — New Endpoints

> **Status:** Draft  
> **Date:** 2026-07-05  
> **Scope:** Subscriptions, Subscribers, Plans enrichment, Subscription Pages, Analytics, Webhook Stats, Payment Pages (Portal/Self-Service)

---

## 1. Amount Convention

**Rule:** All amounts are stored and transmitted in **minor units** (kobo for NGN, cents for USD) as `int`.

| Entity | Stored in DB | Returned by API | Frontend display |
|---|---|---|---|
| `Plan.amount` | `int` (minor) | `Decimal` (major) via validator | Divide by 100 |
| `Invoice.amount_due` | `int` (minor) | `int` (minor) | Divide by 100 |
| `ProrationLineItem.amount_minor` | `int` (minor) | `int` (minor) | Divide by 100 |
| New analytics `total_volume` | `int` (minor) | `int` (minor) | Divide by 100 |
| New analytics `revenue_chart[].amount` | `int` (minor) | `int` (minor) | Divide by 100 |
| Plan `revenue` (new field) | computed | `int` (minor) | Divide by 100 |

**Validation:** Every endpoint that accepts or returns money must be audited to ensure no accidental major/minor mismatch. The `PlanCreate` schema accepts `Decimal` in major units and converts to `int` minor for storage (already done). All other amount fields should be `int` in minor units end-to-end.

---

## 2. Subscription Endpoints

### 2.1 `GET /v1/subscriptions/all`

List all subscriptions, scoped to current tenant and project.

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `limit` | `int` | No | 10 | Max records to return |
| `offset` | `int` | No | 0 | Pagination offset |
| `status` | `str` | No | — | Filter by status (e.g. `active`, `past_due`, `canceled`) |
| `plan_id` | `uuid` | No | — | Filter by plan |

**Response (200):**

```json
{
  "subscriptions": [
    {
      "id": "uuid",
      "tenant_id": "uuid",
      "customer_id": "uuid",
      "plan_id": "uuid",
      "payment_method_id": "uuid | null",
      "status": "active",
      "type": "recurring",
      "current_period_start": "datetime | null",
      "current_period_end": "datetime | null",
      "trial_end": "datetime | null",
      "canceled_at": "datetime | null",
      "cancel_at_period_end": false,
      "created_at": "datetime",
      "updated_at": "datetime",
      "customer": {
        "id": "uuid",
        "name": "string",
        "email": "string"
      },
      "plan": {
        "id": "uuid",
        "name": "string",
        "amount": 10000,
        "currency": "NGN",
        "interval": "monthly"
      }
    }
  ],
  "total_count": 57
}
```

**Notes:**
- Must return `customer_name` / `customer_email` embedded (join on `customers`) so the frontend doesn't need N+1 calls.
- Must return `plan` details (name, amount, currency, interval) embedded.
- Scoped to `tenant_id` and optionally `project_id`.

### 2.2 `GET /v1/subscriptions/{id}`

Get single subscription detail.

**Path Parameters:**

| Parameter | Type | Required |
|---|---|---|
| `id` | `uuid` | Yes |

**Response (200):** `SubscriptionRead` (same as existing, but with embedded `customer` and `plan` objects as above).

**Errors:** `404` — Subscription not found.

### 2.3 `GET /v1/subscriptions/by-plan/{plan_id}`

Get all subscriptions for a particular plan.

**Path Parameters:**

| Parameter | Type | Required |
|---|---|---|
| `plan_id` | `uuid` | Yes |

**Query Parameters:**

| Parameter | Type | Required | Default |
|---|---|---|---|
| `limit` | `int` | No | 10 |
| `offset` | `int` | No | 0 |
| `status` | `str` | No | — |

**Response (200):** Same structure as `/v1/subscriptions/all`.

**Errors:** `404` — Plan not found.

---

## 3. Plans Enrichment

### 3.1 `GET /v1/plans/all` — Enhanced

Current response extended with:

**Additional fields per plan:**

```json
{
  "id": "uuid",
  "name": "string",
  "amount": 10000,
  "currency": "NGN",
  "interval": "monthly",
  "interval_count": 1,
  "trial_period_days": null,
  "installments_count": null,
  "status": "active",
  "created_at": "datetime",
  "updated_at": "datetime",
  "subscription_count": 42,
  "revenue": 4200000
}
```

| New Field | Type | Description |
|---|---|---|
| `subscription_count` | `int` | Number of active subscriptions on this plan |
| `revenue` | `int` | Sum of all successful invoice payments for this plan (minor units) |

### 3.2 `GET /v1/plans/{id}` — Enhanced

Same additional fields as above (`subscription_count`, `revenue`).

---

## 4. Subscribers Endpoints

A "subscriber" is a customer who has at least one subscription.

### 4.1 `GET /v1/subscribers/all`

List all customers who have subscriptions, scoped to current tenant and project.

**Query Parameters:**

| Parameter | Type | Required | Default |
|---|---|---|---|
| `limit` | `int` | No | 10 |
| `offset` | `int` | No | 0 |

**Response (200):**

```json
{
  "subscribers": [
    {
      "customer": {
        "id": "uuid",
        "name": "string",
        "email": "string"
      },
      "subscriptions": [
        {
          "id": "uuid",
          "plan_name": "string",
          "status": "active",
          "amount": 10000,
          "currency": "NGN"
        }
      ],
      "total_subscriptions": 2
    }
  ],
  "total_count": 200
}
```

### 4.2 `GET /v1/subscribers/by-plan/{plan_id}`

List customers subscribed to a specific plan.

**Query Parameters:** Same as above (`limit`, `offset`).

**Response (200):** Same structure.

---

## 5. Subscription Pages (Shareable Checkout Links)

### Concept

A **Subscription Page** is a shareable URL that a merchant can:
- Copy and text to a customer
- Put on a Twitter/X bio
- Embed in their website

When a customer clicks the link, it opens a clean checkout screen (`/subscribe/{code}`) where they enter their name, email, and card to subscribe instantly.

Each Subscription Page maps to exactly one Plan. A unique short code is auto-generated.

### 5.1 `POST /v1/subscription-pages/create`

Create a new subscription page for a plan.

**Request Body:**

```json
{
  "plan_id": "uuid",
  "title": "string | null",
  "description": "string | null"
}
```

| Field | Type | Required | Default |
|---|---|---|---|
| `plan_id` | `uuid` | Yes | — |
| `title` | `str \| None` | No | Plan name |
| `description` | `str \| None` | No | Plan description |

**Response (201):**

```json
{
  "id": "uuid",
  "plan_id": "uuid",
  "code": "abc123",
  "title": "Premium Monthly",
  "description": "Full access to all features",
  "url": "https://orflow.vercel.app/subscribe/abc123",
  "created_at": "datetime",
  "updated_at": "datetime"
}
```

The `code` is a unique, auto-generated short string (e.g. 8 chars, URL-safe).

### 5.2 `GET /v1/subscription-pages/all`

List all subscription pages for the current tenant/project.

**Query Parameters:** `limit`, `offset`.

**Response (200):** Paginated list of subscription pages (including plan name/amount embedded).

### 5.3 `GET /v1/subscription-pages/{id}`

Get a single subscription page.

### 5.4 `GET /v1/subscription-pages/by-plan/{plan_id}`

Get the subscription page for a specific plan.

### 5.5 `PATCH /v1/subscription-pages/{id}/update`

Update title/description of a subscription page.

### 5.6 `DELETE /v1/subscription-pages/{id}`

Delete a subscription page.

### 5.7 `GET /v1/subscription-pages/code/{code}` (Public — No Auth)

Public endpoint that serves a subscription page by its short code. Used by the frontend `/subscribe/{code}` route.

**Response (200):**

```json
{
  "code": "abc123",
  "plan": {
    "name": "Premium Monthly",
    "amount": 10000,
    "currency": "NGN",
    "interval": "monthly",
    "description": "Full access"
  },
  "merchant": {
    "name": "Merchant Name"
  }
}
```

This endpoint does **not** require `X-API-Key`. It's publicly accessible so anyone with the link can see the plan and subscribe.

---

## 6. Analytics Endpoint

### `GET /v1/projects/{projectId}/analytics`

Dashboard landing page analytics.

**Path Parameters:**

| Parameter | Type | Required |
|---|---|---|
| `projectId` | `uuid` | Yes |

**Query Parameters:**

| Parameter | Type | Required | Default |
|---|---|---|---|
| `days` | `int` | No | 30 |

**Response (200):**

```json
{
  "summary": {
    "total_volume": 45000000,
    "active_subscribers": 1240,
    "total_customers": 3150,
    "currency": "NGN"
  },
  "revenue_chart": [
    { "date": "2026-07-01", "amount": 1200000 },
    { "date": "2026-07-02", "amount": 1550000 },
    { "date": "2026-07-03", "amount": 980000 },
    { "date": "2026-07-04", "amount": 2100000 }
  ]
}
```

| Field | Type | Unit | Source |
|---|---|---|---|
| `summary.total_volume` | `int` | Minor (kobo) | SUM of all `paid` invoice amounts in this project |
| `summary.active_subscribers` | `int` | Count | COUNT of unique subscriptions with status `active` |
| `summary.total_customers` | `int` | Count | COUNT of unique customers under this tenant |
| `summary.currency` | `str` | — | Project currency (default: NGN) |
| `revenue_chart[].date` | `str` | YYYY-MM-DD | Date |
| `revenue_chart[].amount` | `int` | Minor (kobo) | Sum of paid invoice amounts on that date |

**Scoping:** Must be scoped to both `tenant_id` (from auth context) and `project_id` (from path param). Returns `403` if the project doesn't belong to the current tenant.

---

## 7. Webhook Stats

### `GET /v1/webhooks/events/stats`

Dashboard widget showing webhook delivery health.

**Response (200):**

```json
{
  "total": 150,
  "successful": 120,
  "failed": 10,
  "pending": 20
}
```

| Field | Type | Source |
|---|---|---|
| `total` | `int` | Total webhook events sent |
| `successful` | `int` | Events where `status == "delivered"` |
| `failed` | `int` | Events where `status == "failed"` |
| `pending` | `int` | Events where `status == "pending"` or `"retrying"` |

**Scoping:** Must be scoped to `tenant_id` and `project_id`.

---

## 8. Self-Service Portal

### 8.1 What It Is

The **Self-Service Portal** is a customer-facing web page where end customers (subscribers) can manage their own subscriptions without contacting the merchant.

**URL:** `https://orflow.vercel.app/portal/{token}`

### 8.2 Who It's For

End customers of any merchant who uses Orflow for subscription billing. The merchant's subscribers can:

- View their current plan, price, and next charge date
- Update their payment method (credit card)
- View billing history (paid/failed invoices)
- Cancel, pause, or resume their subscription
- See status-specific banners (past-due, suspended)

### 8.3 How Merchants Use It

**Step 1:** Merchant creates a subscription via `POST /v1/subscriptions/create`

**Step 2:** Merchant generates a portal access token:

```
POST /v1/portal/generate-token
X-API-Key: sk_live_abc123
{ "subscription_id": "..." }

→ { "token": "eyJhbGciOiJIUzI1NiIs..." }
```

**Step 3:** Merchant puts a link in their app:

```
https://orflow.vercel.app/portal/eyJhbGciOiJIUzI1NiIs...
```

Or embeds it in an iframe:

```html
<iframe src="https://orflow.vercel.app/portal/eyJhbGciOiJIUzI1NiIs..." />
```

**Step 4:** Customer clicks the link → lands on Orflow's portal page.

**The merchant does NOT need to build:**
- Any UI for billing management
- Card entry forms (PCI scope)
- Dunning / retry / recovery flows
- Invoice history views

### 8.4 Portal Auth Mechanism

Portal endpoints use **signed JWT tokens** instead of `X-API-Key`:

| Endpoint | Method | Auth | Purpose |
|---|---|---|---|
| `/v1/portal/generate-token` | POST | `sk_` key | Merchant generates JWT for a subscription |
| `/v1/portal/subscription` | GET | JWT | Get subscription details for portal display |
| `/v1/portal/invoices` | GET | JWT | Billing history |
| `/v1/portal/payment-method` | GET | JWT | Masked card details |
| `/v1/portal/checkout` | POST | JWT | Initiate Nomba checkout for card update |
| `/v1/portal/cancel` | POST | JWT | Cancel subscription |
| `/v1/portal/pause` | POST | JWT | Pause subscription |
| `/v1/portal/resume` | POST | JWT | Resume subscription |

**JWT contains:** `subscription_id`, `customer_id`, `tenant_id`, `exp` (24h expiry).

**JWT is signed with** a server-side secret (not per-merchant to start).

### 8.5 Portal Endpoints

#### `POST /v1/portal/generate-token`

**Request:**
```json
{
  "subscription_id": "uuid"
}
```

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "expires_at": "2026-07-06T01:00:00Z"
}
```

#### `GET /v1/portal/subscription`

**Headers:** `Authorization: Bearer {token}`

**Response:** `SubscriptionRead` with embedded `customer` (name, email) and `plan` (name, amount, currency, interval).

#### `GET /v1/portal/invoices`

**Query:** `?limit=10&offset=0`

**Response:** Paginated list of invoices for this subscription.

#### `GET /v1/portal/payment-method`

**Response:**
```json
{
  "id": "uuid",
  "type": "card",
  "last_four": "4242",
  "expiry_month": 12,
  "expiry_year": 2027,
  "is_default": true
}
```

#### `POST /v1/portal/checkout`

Initiates a new Nomba checkout session to re-tokenize a new card.

**Response:**
```json
{
  "checkout_link": "https://nomba-checkout.com/...",
  "order_reference": "ref_123"
}
```

Frontend redirects customer to Nomba. After successful payment, Nomba sends a webhook with the new token, and the backend updates the payment method.

#### `POST /v1/portal/cancel`

**Response:** `SubscriptionRead` (updated).

#### `POST /v1/portal/pause`

**Response:** `SubscriptionRead`.

#### `POST /v1/portal/resume`

**Response:** `SubscriptionRead`.

---

## 9. Implementation Order

### Phase 1 — Merchant Dashboard Data
1. Enhance `GET /v1/plans/all` and `GET /v1/plans/{id}` with `subscription_count` and `revenue`
2. `GET /v1/subscriptions/all` with filters, pagination, embedded customer/plan
3. `GET /v1/subscriptions/{id}` with embedded customer/plan
4. `GET /v1/subscriptions/by-plan/{plan_id}`
5. `GET /v1/subscribers/all` and `GET /v1/subscribers/by-plan/{plan_id}`
6. `GET /v1/projects/{projectId}/analytics`
7. `GET /v1/webhooks/events/stats`

### Phase 2 — Subscription Pages
8. Subscription pages CRUD (create, list, get, update, delete)
9. Public `GET /v1/subscription-pages/code/{code}` endpoint

### Phase 3 — Self-Service Portal
10. Portal JWT auth mechanism
11. Portal endpoints (subscription, invoices, payment-method, checkout, cancel, pause, resume)
12. Frontend portal routes ported from `test/` to `frontend/src/`

### Phase 4 — Amount Audit
13. Audit every endpoint for major/minor unit consistency
14. Add explicit field documentation (comments) for all amount fields

---

## 10. New Router Files Needed

| File | Purpose |
|---|---|
| `backend/app/subscriptions_v2/router.py` | Extended subscription endpoints (list, by-plan, enhanced get) |
| `backend/app/subscribers/router.py` | Subscriber endpoints |
| `backend/app/subscription_pages/router.py` + `schemas.py` + `models.py` + `service.py` | Subscription pages CRUD |
| `backend/app/analytics/router.py` + `service.py` | Analytics endpoint |
| `backend/app/webhooks/stats_router.py` | Webhook stats endpoint |
| `backend/app/portal/router.py` + `schemas.py` + `service.py` | Self-service portal endpoints |

Or these can be added to existing routers where appropriate (e.g. subscription endpoints in `app/subscriptions/router.py`).

---

## 11. Existing Files That Need Changes

| File | Change |
|---|---|
| `app/plans/schemas.py` — `PlanRead` | Add `subscription_count: int` and `revenue: int` |
| `app/plans/service.py` or `router.py` | Compute aggregates via subquery/join |
| `app/subscriptions/router.py` | Add `GET /all`, `GET /{id}`, `GET /by-plan/{plan_id}` endpoints |
| `app/subscriptions/schemas.py` | Add response models that embed customer + plan |
| `main.py` | Mount new routers |
| `app/core/middleware.py` | Add `/v1/subscription-pages/code/` and `/v1/portal/*` (public/portal endpoints) to exempt list where needed |
