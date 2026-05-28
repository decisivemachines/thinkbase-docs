# Thinkbase Platform — B2B Data API

Product documentation for the Thinkbase B2B Data API. This is the single source of truth for the API architecture, endpoints, billing, admin tooling, and infrastructure.

## Architecture Overview

```
┌─────────────────┐     ┌──────────────┐     ┌──────────────────┐
│  Partner App     │────▶│  b2b-api     │────▶│  Supabase        │
│  (X-API-Key)     │     │  port 3200   │     │  (Postgres)      │
└─────────────────┘     │              │     │                  │
                        │  ┌─────────┐ │     │  api_keys        │
                        │  │ apiKey  │ │────▶│  api_usage_logs  │
                        │  │ Auth    │ │     │  debates          │
                        │  ├─────────┤ │     │  arguments        │
                        │  │ usage   │ │     │  answer_options   │
                        │  │ Tracker │─┼────▶│  prediction_snapshots │
                        │  └─────────┘ │     │  categories       │
                        │       │      │     └──────────────────┘
                        │       ▼      │
                        │    Stripe    │     ┌──────────────────┐
                        │   (meter     │────▶│  Stripe          │
                        │    events)   │     │  Billing Meters  │
                        └──────────────┘     │  Subscriptions   │
                                             │  Invoices        │
┌─────────────────┐     ┌──────────────┐     └──────────────────┘
│  Admin Dashboard │────▶│  admin-api   │
│  (React + Vite)  │     │  port 3100   │
│  admin/          │     │              │
└─────────────────┘     └──────────────┘
```

**Services:**

| Service | Directory | Port | Purpose |
|---------|-----------|------|---------|
| B2B Data API | `b2b-api/` | 3200 | Public REST API for partners |
| Admin API | `admin-api/` | 3100 | Internal admin operations |
| Admin Dashboard | `admin/` | 5173 (dev) | React UI for managing partners |
| Consumer Backend | `backend/` | 3000 | Mobile/web app backend (existing) |

---

## B2B Data API (`b2b-api/`)

### Endpoints

#### Public (no auth)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1` | API info, endpoint list, tier descriptions |
| GET | `/v1/health` | Health check (`{ status: 'ok', version, timestamp }`) |

#### Free Tier

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/debates` | List debates with options + voting percentages |
| GET | `/v1/debates/:slug` | Single debate detail |
| GET | `/v1/categories` | All categories with debate counts |
| GET | `/v1/trending` | Top debates by trending score (open/closing_soon only) |

#### Tier 1+ (and above)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/debates/:slug/arguments` | Arguments with position, upvotes — the "why" behind opinions |
| GET | `/v1/debates/:slug/history` | Prediction percentage snapshots over time |
| GET | `/v1/search?q=` | Text search across debate titles and descriptions |

#### Webhooks (Stripe)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/webhooks/stripe` | Stripe billing webhook (signature-verified) |

### Query Parameters

**Debates list** (`GET /v1/debates`):

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | int | 1 | Page number |
| `limit` | int | 50 | Results per page (max varies by tier) |
| `category` | string | — | Filter by category slug (e.g., `politics`) |
| `status` | string | — | Filter by status: `open`, `closed`, `resolved`, `closing_soon` |
| `sort` | string | `recent` | Sort: `recent`, `popular`, `ending_soon` |

**Arguments** (`GET /v1/debates/:slug/arguments`):

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | int | 1 | Page number |
| `limit` | int | 50 | Results per page |
| `option_id` | uuid | — | Filter to a specific answer option |
| `sort` | string | `top` | Sort: `top` (upvotes) or `recent` |

**History** (`GET /v1/debates/:slug/history`):

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `from` | ISO 8601 | — | Start of date range |
| `to` | ISO 8601 | — | End of date range |

**Search** (`GET /v1/search`):

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `q` | string | required | Search query (sanitized: commas, parens, dots stripped) |
| `category` | string | — | Filter by category slug |
| `status` | string | — | Filter by status |
| `page` | int | 1 | Page number |
| `limit` | int | 50 | Results per page |

### Response Shapes

**Paginated list** (debates, arguments, search):
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total": 1648,
    "has_more": true
  }
}
```

**Single debate:**
```json
{
  "data": {
    "id": "uuid",
    "title": "Should the US ban TikTok?",
    "slug": "should-the-us-ban-tiktok",
    "description": "...",
    "category": "Politics",
    "category_slug": "politics",
    "status": "open",
    "total_predictions": 142,
    "options": [
      {
        "id": "uuid",
        "text": "Yes",
        "emoji": "✅",
        "vote_percentage": 62.5,
        "predictor_count": 89,
        "display_order": 0
      }
    ],
    "starts_at": "2026-05-01T00:00:00Z",
    "ends_at": "2026-06-01T00:00:00Z",
    "created_at": "2026-05-01T00:00:00Z"
  }
}
```

**Argument:**
```json
{
  "id": "uuid",
  "position": "Yes",
  "position_emoji": "✅",
  "body": "The real question isn't whether to ban it...",
  "upvote_count": 89,
  "created_at": "2026-05-02T14:30:00Z"
}
```

**History snapshot:**
```json
{
  "option_id": "uuid",
  "option_text": "Yes",
  "predictor_count": 89,
  "prediction_percentage": 62.5,
  "captured_at": "2026-05-15T08:00:00Z"
}
```

### Authentication

Partners authenticate via the `X-API-Key` header:

```bash
curl -H "X-API-Key: tb_live_abc123..." https://api.trythinkbase.com/v1/debates
```

**Key format:** `tb_live_` + 32 random hex characters (40 chars total).

**How it works:**
1. Partner sends raw key in `X-API-Key` header
2. `apiKeyAuth` middleware hashes it with SHA-256
3. Calls `authenticate_and_increment` Postgres RPC in one round-trip:
   - Validates hash against `api_keys.key_hash` (WHERE `is_active = true`)
   - Handles daily/monthly counter rollover in UTC
   - Atomically increments counters with `FOR UPDATE` row lock
   - Returns key metadata + rate limit status
4. If over daily limit, returns 429 with rate limit headers
5. Attaches key info to `req.apiKey` for downstream middleware

**Rate limit headers** (on every authenticated response):
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 842
X-RateLimit-Used-Monthly: 4523
```

### Tier System

Usage-based pricing at **$0.003/call** with no base fees. Tiers auto-graduate based on lifetime spend.

| Tier | Spend Threshold | Daily Limit | Max Page Size | Per-Call Rate | Unlocks |
|------|----------------|-------------|---------------|---------------|---------|
| **free** | $0 | 100 | 100 | $0.003 | Debates, categories, trending, live percentages |
| **tier_1** | $5 | 1,000 | 250 | $0.003 | + Arguments, search, historical snapshots |
| **tier_2** | $50 | 10,000 | 500 | $0.003 | Higher rate limits |
| **tier_3** | $250 | 100,000 | 1,000 | $0.003 | Production-scale access |
| **tier_4** | $1,000 | 1,000,000 | 1,000 | $0.003 | Maximum throughput |

Higher tiers inherit all lower-tier permissions. Page size is clamped server-side via `clampLimit()` in `types/index.ts`. Tier graduation happens automatically when cumulative spend crosses a threshold (triggered by `invoice.paid` webhook).

### Middleware Pipeline

Every authenticated request goes through this chain:

```
Request → apiKeyAuth → usageTracker → router → [requireTier per-route] → handler → Response
           │                │
           │                ├─ INSERT api_usage_logs (async, on res.finish)
           │                └─ Stripe meterEvents.create (async, if stripe_customer_id)
           │
           └─ authenticate_and_increment RPC (1 DB call: auth + rate limit)
```

`requireTier()` is applied inside the router at individual route level (e.g., `router.get('/:slug/arguments', requireTier('tier_1'), handler)`), not as a top-level middleware. This means usage is logged even for requests that are subsequently rejected by tier gating.

**Key design decisions:**
- Auth + rate limiting in a single DB round-trip (atomic, no race conditions)
- Usage tracking is fire-and-forget (runs on `res.finish` event, doesn't add latency)
- Stripe meter events include idempotency `identifier` (UUID) and explicit `timestamp`
- Meter events retry up to 2 times on 429 with exponential backoff + jitter

---

## Stripe Billing

### Overview

All API calls are billed at a flat **$0.003/call** via a single Stripe metered price. There are no base fees. Tiers auto-graduate based on lifetime spend.

Invoices are sent (not auto-charged) with 30-day payment terms (`collection_method: 'send_invoice'`).

### Stripe Objects

| Object | ID Pattern | Description |
|--------|-----------|-------------|
| Billing Meter | `mtr_...` | Counts `thinkbase_api_call` events, sum aggregation, maps by `stripe_customer_id` |
| Product | `prod_...` | "Thinkbase Data API" |
| Metered Price | `price_...` | $0.003/call, linked to billing meter (env: `STRIPE_PRICE_METERED`) |
| Customer | `cus_...` | One per partner, created when payment method is added |
| Subscription | `sub_...` | Single item: metered price only |

### Billing Lifecycle

```
Partner adds payment method
  → Stripe Customer created
  → Stripe Subscription created (single metered price, send_invoice, 30d due)
  → stripe_customer_id + stripe_subscription_id stored on api_keys row

Partner makes API calls
  → usageTracker fires stripe.billing.meterEvents.create() per request
  → identifier (UUID) prevents double-counting on retries
  → Only fires for partners with stripe_customer_id (free tier skipped)

End of billing period
  → Stripe auto-generates invoice (metered usage only, no base fee)
  → Invoice sent to partner email
  → Webhook: invoice.created → we return 200 (prevents 72hr delay)
  → Webhook: invoice.paid → check lifetime spend, auto-graduate tier if threshold crossed
  → Webhook: invoice.payment_failed → logged (future: suspend access)

Auto-graduation (on invoice.paid)
  → Sum all paid invoices for the customer
  → If lifetime spend >= next tier threshold → update api_keys.tier + rate_limit_daily
  → Thresholds: tier_1=$5, tier_2=$50, tier_3=$250, tier_4=$1,000

Key revocation (via admin POST /partners/:id/revoke)
  → Cancel Stripe Subscription
  → Set is_active = false, stripe_subscription_id = null
```

### Webhook Handler

Mounted at `POST /webhooks/stripe` with `express.raw()` for signature verification.

| Event | Action |
|-------|--------|
| `invoice.created` | Return 200 immediately (prevents Stripe's 72hr finalization delay) |
| `invoice.paid` | Log payment amount, check lifetime spend, auto-graduate tier if threshold crossed |
| `invoice.payment_failed` | Log failure |
| `customer.subscription.updated` | Log status, warn on `past_due`/`unpaid`, sync DB on `canceled` |
| `customer.subscription.deleted` | Clear `stripe_subscription_id` from `api_keys` |

For valid requests that pass signature verification, handler errors are caught internally and the endpoint returns `{ received: true }` (200) to prevent Stripe retries (up to 3 days of exponential backoff). Invalid signatures return 400, and requests when Stripe is not configured return 503.

### Setup

Run once after setting `STRIPE_SECRET_KEY` in `.env`:

```bash
cd b2b-api && npx tsx src/scripts/stripe-setup.ts
```

Creates: Billing Meter + Product + 1 metered Price ($0.003/call). Outputs env vars to add:

```
STRIPE_METER_EVENT_NAME=thinkbase_api_call
STRIPE_PRODUCT_ID=prod_...
STRIPE_PRICE_METERED=price_...
```

Also add `STRIPE_WEBHOOK_SECRET=whsec_...` from the Stripe Dashboard (Developers > Webhooks).

---

## Admin Dashboard — Partner Management

### Navigation

Located under the **B2B** section in the sidebar (`admin/src/components/AdminShell.tsx`), route: `#/partners`, page title: "API Partners".

### Features

**Stats row** (top of page):
- Total Keys — count of all API keys
- Active — count where `is_active = true`
- Calls Today — sum of `calls_today` across all keys
- Calls This Month — sum of `calls_this_month` across all keys

**Toolbar:**
- Search input (filters by partner name, email, key prefix)
- Tier dropdown filter (all / free / tier_1 / tier_2 / tier_3 / tier_4)
- "+ Create API Key" button

**Table columns:**
- Partner (name + email)
- Key Prefix (monospace, e.g., `tb_live_abc1234...`)
- Tier (color-coded badge: free=gray, tier_1=blue, tier_2=amber, tier_3=green, tier_4=purple)
- Today (daily call count)
- Month (monthly call count)
- Status (Active=green, Revoked=red)
- Actions (Usage toggle, Revoke button)

**Usage panel** (expands inline per partner):
- Total calls (last 30 days)
- Average response time (ms)
- Top 5 endpoints by call count
- Daily call bar chart (last 30 days)
- Data from `get_partner_usage_stats` RPC (server-side aggregation, no row limits)

**Create Key modal:**
- Partner Name (required)
- Contact Email (required)
- Tier dropdown (defaults to free)
- On submit: calls `POST /partners`, returns raw key

**Key Reveal modal** (shown once after creation):
- Displays raw API key in monospace
- Copy button with "Copied" feedback
- Warning: "Copy it now — it won't be shown again"

**Revoke flow:**
- Confirmation dialog
- Calls `POST /partners/:id/revoke`
- Cancels Stripe subscription server-side
- Refreshes list

### Admin API Endpoints

All routes require `adminAuth` middleware (Supabase JWT + email allowlist).

| Method | Path | Description |
|--------|------|-------------|
| GET | `/partners` | List keys (page, limit, tier, search) |
| POST | `/partners` | Create key (generates raw key, hashes, creates Stripe customer + metered subscription if payment method provided) |
| PATCH | `/partners/:id` | Update partner info (tier auto-graduates via spend, manual override available) |
| POST | `/partners/:id/revoke` | Revoke key (cancels Stripe subscription) |
| GET | `/partners/:id/usage` | Usage stats for last N days (via `get_partner_usage_stats` RPC) |

---

## Database Schema

### `api_keys`

| Column | Type | Default | Nullable | Description |
|--------|------|---------|----------|-------------|
| `id` | uuid | `gen_random_uuid()` | NO | Primary key |
| `key_hash` | text | — | NO | SHA-256 hex of raw API key (UNIQUE) |
| `key_prefix` | text | — | NO | First 15 chars for display (`tb_live_xxxxxxx`) |
| `partner_name` | text | — | NO | Company/organization name |
| `contact_email` | text | — | NO | Partner contact email |
| `tier` | text | `'free'` | NO | CHECK: `free`, `tier_1`, `tier_2`, `tier_3`, `tier_4` |
| `rate_limit_daily` | int | `100` | NO | Max API calls per day |
| `calls_today` | int | `0` | NO | Counter (reset at UTC midnight by RPC) |
| `calls_today_reset_at` | timestamptz | `now()` | NO | Last daily reset |
| `calls_this_month` | int | `0` | NO | Counter (reset at month start by RPC) |
| `calls_month_reset_at` | timestamptz | `date_trunc('month', now())` | NO | Last monthly reset |
| `is_active` | boolean | `true` | NO | `false` when revoked |
| `revoked_at` | timestamptz | — | YES | When key was revoked |
| `last_used_at` | timestamptz | — | YES | Updated by `authenticate_and_increment` |
| `created_at` | timestamptz | `now()` | NO | — |
| `updated_at` | timestamptz | `now()` | NO | — |
| `stripe_customer_id` | text | — | YES | Stripe customer for billing |
| `stripe_subscription_id` | text | — | YES | Stripe subscription for billing |

**Indexes:**
- `idx_api_keys_key_hash` — btree on `key_hash` WHERE `is_active = true` (auth hot path)
- `idx_api_keys_stripe_customer` — btree on `stripe_customer_id` WHERE NOT NULL

### `api_usage_logs`

| Column | Type | Default | Nullable | Description |
|--------|------|---------|----------|-------------|
| `id` | bigint | GENERATED ALWAYS AS IDENTITY | NO | Primary key |
| `api_key_id` | uuid | — | NO | FK → `api_keys.id` |
| `endpoint` | text | — | NO | Request path (e.g., `/debates`) |
| `method` | text | `'GET'` | NO | HTTP method |
| `status_code` | int | — | NO | Response status code |
| `response_ms` | int | — | YES | Latency in milliseconds |
| `created_at` | timestamptz | `now()` | NO | — |

**Indexes:**
- `idx_api_usage_logs_key_time` — btree on `(api_key_id, created_at DESC)` — per-key billing queries
- `idx_api_usage_logs_created` — btree on `created_at DESC` — global analytics

### Postgres RPCs

| Function | Purpose | Called From |
|----------|---------|------------|
| `authenticate_and_increment(p_key_hash)` | Auth + rate limit in 1 atomic call. Validates key, handles daily/monthly counter rollover in UTC, increments with FOR UPDATE lock. Returns key metadata + counters + `stripe_customer_id`. | b2b-api `apiKeyAuth` middleware |
| `get_debate_arguments(p_slug, p_option_id, p_sort, p_limit, p_offset)` | Fetches arguments by debate slug with join, sort, pagination. Eliminates slug-lookup round-trip. | b2b-api `GET /:slug/arguments` |
| `get_debate_history(p_slug, p_from, p_to)` | Fetches prediction snapshots with option text join. Eliminates 3 round-trips → 1. | b2b-api `GET /:slug/history` |
| `get_category_counts()` | Server-side GROUP BY debate counts per category. Eliminates fetching all debate rows. | b2b-api `GET /v1/categories` |
| `get_partner_usage_stats(p_key_id, p_since)` | Server-side aggregation of usage logs. Returns total_calls, avg_response_ms, daily_counts (JSONB), endpoint_counts (JSONB), status_counts (JSONB). No row limits. | admin-api `GET /partners/:id/usage` |

### Custom Indexes (added for B2B API)

| Index | Table | Definition | Purpose |
|-------|-------|-----------|---------|
| `idx_debates_description_trgm` | debates | GIN trigram on `description` | Search `ilike` on description |
| `idx_debates_total_predictions_desc` | debates | btree on `total_predictions DESC` WHERE `status != 'content_seed'` | `?sort=popular` queries |

---

## Performance Characteristics

### DB Calls Per Request

| Route | Auth | Route Logic | Async | Total |
|-------|------|-------------|-------|-------|
| `GET /v1/debates` | 1 (RPC) | 1 (query) | 1 (usage log) + 1 (meter event) | 3-4 |
| `GET /v1/debates/:slug` | 1 | 1 | 1-2 | 3-4 |
| `GET /v1/debates/:slug/arguments` | 1 | 1 (RPC) | 1-2 | 3-4 |
| `GET /v1/debates/:slug/history` | 1 | 1 (RPC) | 1-2 | 3-4 |
| `GET /v1/categories` | 1 | 2 (parallel) | 1-2 | 4-5 |
| `GET /v1/trending` | 1 | 1 | 1-2 | 3-4 |
| `GET /v1/search` | 1 | 1 | 1-2 | 3-4 |

Async calls (usage log + Stripe meter event) fire after response is sent and don't add latency.

---

## Environment Variables

### b2b-api `.env`

```
# Required
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_KEY=eyJ...

# Optional
PORT=3200
NODE_ENV=development
CORS_ORIGIN=

# Stripe (required for billing — meter event reporting)
STRIPE_SECRET_KEY=sk_live_...
STRIPE_METER_EVENT_NAME=thinkbase_api_call
# Note: STRIPE_WEBHOOK_SECRET is read directly from process.env in webhooks.ts,
# not via the centralized env.ts config.
STRIPE_WEBHOOK_SECRET=whsec_...
```

### admin-api `.env`

```
# Core (existing — see admin-api/src/config/env.ts for full list)
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJ...
SUPABASE_ANON_KEY=eyJ...
ADMIN_EMAILS=admin@trythinkbase.com

# Stripe (required for partner billing — subscription management)
STRIPE_SECRET_KEY=sk_live_...

# Stripe metered price (from stripe-setup script — only needed in admin-api)
STRIPE_PRICE_METERED=price_...
```

---

## E2E Test Suite

Located at `b2b-api/src/test.ts`. Run with:

```bash
cd b2b-api
PORT=3200 npx tsx src/index.ts &   # start server
npx tsx src/test.ts                 # run tests
```

**103 assertions** covering:
- Public endpoints (health, API info)
- Auth middleware (missing key → 401, invalid key → 401, valid key → 200)
- Tier gating (tier_4 key accesses tier_1 endpoints)
- Debates list (pagination, sorting, category/status filters, response shape)
- Single debate (slug lookup, 404 handling)
- Arguments (pagination, sorting, 404 handling, response shape)
- History (chronological order, date range filtering, 404 handling)
- Categories (debate counts, known slugs present)
- Trending (only open/closing_soon, category filter, limit)
- Search (required `q` param, empty `q` → 400, filters, pagination)
- Rate limit headers (all three present and numeric)
- Usage logging (middleware doesn't crash the pipeline)

Test API key: `tb_test_e2e_runner_key` (tier_4, 1M daily limit).

---

## API Docs

**Platform:** Mintlify (free Hobby tier)
**Domain:** `docs.trythinkbase.com`
**Repo:** [decisivemachines/thinkbase-docs](https://github.com/decisivemachines/thinkbase-docs) (public)
**Local:** `~/Developer/thinkbase-docs/`

Built and ready to deploy:

```
thinkbase-docs/
├── docs.json               # Mintlify config (theme: mint, nav, colors, API playground)
├── openapi.yaml            # OpenAPI 3.1.0 spec — all 7 endpoints with full schemas
└── guides/
    ├── introduction.mdx    # Quick start, base URL, response format
    ├── authentication.mdx  # X-API-Key header, key format, error responses
    ├── rate-limits.mdx     # Limits by tier, headers, 429 handling
    └── tiers.mdx           # Pricing table, what each tier unlocks, billing
```

**OpenAPI spec includes:** All endpoints, query params with defaults, complete response schemas (Debate, Option, Argument, Snapshot, Category, Pagination), separate error schemas for 401 (AuthError with `docs` field), 403 (TierError with `current_tier`, `required_tier`, `upgrade_url`), and 429 (RateLimitError with `limit`, `upgrade_url`). All authenticated endpoints document 401 and 429 responses.

**To deploy:**
1. Repo is live at [decisivemachines/thinkbase-docs](https://github.com/decisivemachines/thinkbase-docs) (public, under org)
2. Connect at [mintlify.com/start](https://mintlify.com/start), select `decisivemachines/thinkbase-docs`
3. Add CNAME: `docs` → `cname.vercel-dns.com`
4. Add logo SVGs at `logo/light.svg` and `logo/dark.svg` (referenced in docs.json but not yet created)

Mintlify auto-generates API Reference pages from the OpenAPI spec with an interactive playground where partners paste their key and test live.

---

## Data We Sell

| Dataset | Volume | Tier | Unique Value |
|---------|--------|------|-------------|
| Debates with voting percentages | 1,648 debates, 4,240 options | Free | Real-time crowd opinion on current topics |
| Structured arguments | 40,224 arguments with position tags | Tier 1+ | The "why" behind opinions — no one else has this |
| Prediction snapshots | 8,225 snapshots across 1,646 debates | Tier 1+ | Opinion trends over time |
| Categories | 14 categories, Politics leads (22K+ debates) | Free | Organized topic taxonomy |

The differentiator vs. prediction markets (Polymarket, Kalshi): they capture **what** people think (probabilities). We capture **why** — structured arguments tied to positions with upvote signals.
