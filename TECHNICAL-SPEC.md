# AI Guest Agent SaaS — Technical Specification

**Document Version:** 1.0
**Date:** March 27, 2026
**Author:** CTO, DLJ Properties
**Classification:** Internal / Confidential
**Status:** Draft for CEO Review

---

## Executive Summary

This document specifies the architecture, onboarding flow, competitive moat, billing model, customer dashboard, infrastructure scaling plan, and security posture for productizing the DLJ Properties AI Guest Agent as a multi-tenant SaaS product.

**What exists today:** A webhook server + AI customer service agent running on local Ollama infrastructure (Mac Studio 122B + Mac Mini 35B) that has sent **442+ automated guest replies** across **7 properties** with **under 2-minute response times** and **zero escalation failures**.

**What we are building:** A hosted, multi-tenant SaaS product that lets any Guesty host connect their account and get an AI agent that learns their hosting style and responds to guests automatically.

**Target:** First beta customer live in **8 weeks**. Full MVP in **10-14 weeks**.

---

## Table of Contents

1. [Multi-Tenant Architecture](#1-multi-tenant-architecture)
2. [Onboarding Flow](#2-onboarding-flow)
3. [Host-Reply-Learner (The Moat)](#3-host-reply-learner-the-moat)
4. [Billing — Stripe Integration](#4-billing--stripe-integration)
5. [Customer Dashboard](#5-customer-dashboard)
6. [Infrastructure & Scaling](#6-infrastructure--scaling)
7. [Security](#7-security)
8. [Summary & Roadmap](#8-summary--roadmap)

---

## 1. Multi-Tenant Architecture

**Build Estimate:** 3-4 weeks

### Design Decision: Shared Infrastructure with Tenant Isolation

We recommend **shared infrastructure with strict tenant isolation** over per-customer instances. Per-customer instances (one server per host) are operationally expensive, wasteful at low utilization, and do not scale past ~20 customers without a dedicated DevOps team. Shared infrastructure with proper isolation gives us the economics of multi-tenancy with the security guarantees of separation.

### What Each Tenant Gets

| Resource | Isolation Method |
|----------|-----------------|
| Message queue | Dedicated queue per tenant (namespaced) |
| Knowledge base | Isolated namespace in vector DB |
| Guesty API credentials | Encrypted per-tenant, dedicated secrets table |
| Configuration | Row-level security in PostgreSQL |
| Webhook endpoint | Single ingress, routed by tenant ID |
| AI agent context | Per-tenant system prompt + RAG namespace |

### Architecture Diagram

```
                          INBOUND GUEST MESSAGES
                                  |
                                  v
                    +---------------------------+
                    |      API GATEWAY          |
                    |  (rate limiting, auth,    |
                    |   TLS termination)        |
                    +---------------------------+
                                  |
                                  v
                    +---------------------------+
                    |      TENANT ROUTER        |
                    |  webhook payload -->      |
                    |  extract tenant_id -->    |
                    |  validate --> route       |
                    +---------------------------+
                                  |
                   +--------------+--------------+
                   |              |              |
                   v              v              v
            +----------+   +----------+   +----------+
            | Queue T1 |   | Queue T2 |   | Queue TN |
            | (tenant) |   | (tenant) |   | (tenant) |
            +----------+   +----------+   +----------+
                   |              |              |
                   +--------------+--------------+
                                  |
                                  v
                    +---------------------------+
                    |     AI AGENT POOL         |
                    |  (shared compute,         |
                    |   per-tenant context)     |
                    |                           |
                    |  1. Load tenant config    |
                    |  2. Query tenant KB (RAG) |
                    |  3. Generate reply        |
                    |  4. Send via tenant creds |
                    +---------------------------+
                                  |
                   +--------------+--------------+
                   |              |              |
                   v              v              v
            +-----------+  +-----------+  +-----------+
            | Guesty    |  | Guesty    |  | Guesty    |
            | API (T1   |  | API (T2   |  | API (TN   |
            | creds)    |  | creds)    |  | creds)    |
            +-----------+  +-----------+  +-----------+
```

### Data Model (PostgreSQL)

```
tenants
  id              UUID PRIMARY KEY
  name            TEXT
  email           TEXT UNIQUE
  plan            ENUM('trial','starter','pro','business')
  stripe_customer TEXT
  created_at      TIMESTAMPTZ
  status          ENUM('active','suspended','cancelled')

tenant_guesty_credentials
  tenant_id       UUID REFERENCES tenants(id)
  access_token    BYTEA          -- AES-256-GCM encrypted
  refresh_token   BYTEA          -- AES-256-GCM encrypted
  token_expires   TIMESTAMPTZ
  iv              BYTEA          -- initialization vector
  created_at      TIMESTAMPTZ

tenant_properties
  id              UUID PRIMARY KEY
  tenant_id       UUID REFERENCES tenants(id)
  guesty_listing  TEXT           -- Guesty listing ID
  nickname        TEXT
  address         TEXT
  active          BOOLEAN DEFAULT true
  kb_namespace    TEXT           -- vector DB namespace

messages
  id              UUID PRIMARY KEY
  tenant_id       UUID REFERENCES tenants(id)
  property_id     UUID REFERENCES tenant_properties(id)
  reservation_id  TEXT
  direction       ENUM('inbound','outbound')
  content         TEXT
  ai_generated    BOOLEAN
  host_edited     BOOLEAN DEFAULT false
  response_time   INTEGER        -- milliseconds
  created_at      TIMESTAMPTZ
```

### Queue Technology

- **MVP:** BullMQ (Redis-backed, Node.js native, battle-tested)
- **Scale:** Migrate to SQS or Cloud Tasks if Redis becomes a bottleneck (unlikely before 500 tenants)
- Each tenant gets a named queue: `messages:tenant:{tenant_id}`
- Queue guarantees: at-least-once delivery, dead letter queue for failures, 3 retries with exponential backoff

### Tenant Routing Logic

```
INCOMING WEBHOOK
  --> Parse payload (Guesty webhook format)
  --> Extract accountId from payload (Guesty account identifier)
  --> Lookup tenant by guesty_account_id in tenants table
  --> If not found: log + discard (possible stale webhook)
  --> If found: validate tenant status == 'active'
  --> If active: enqueue to tenant's message queue
  --> If suspended/cancelled: log + discard
```

---

## 2. Onboarding Flow

**Build Estimate:** 3-4 weeks
**Target:** Signup to live agent in under 30 minutes

### Step-by-Step Flow

```
  SIGNUP           CONNECT          DISCOVER         BUILD KB         GO LIVE
+--------+      +--------+      +--------+       +--------+      +--------+
| Email  | ---> | Guesty | ---> | Pull   | --->  | Index  | ---> | Webhook|
| + Pass |      | OAuth2 |      | All    |       | Props  |      | Active |
|        |      | Flow   |      | Data   |       | + Msgs |      |        |
+--------+      +--------+      +--------+       +--------+      +--------+
  ~1 min          ~2 min          ~5 min           ~10 min         ~1 min
                                                                  --------
                                                            TOTAL: ~19 min
```

### Step 1: Signup (1 minute)

- Landing page with email + password form
- Auth provider: **Clerk** (fastest to integrate, good free tier, handles MFA)
- On signup: create `tenants` row, send verification email
- No credit card required (14-day free trial)

### Step 2: Connect Guesty — OAuth2 Flow (2 minutes)

- Customer clicks "Connect Guesty" button
- Redirected to Guesty OAuth2 authorization page
- Customer authorizes our app to access their Guesty account
- Guesty redirects back with authorization code
- Backend exchanges code for access + refresh tokens
- Tokens encrypted with AES-256-GCM and stored in `tenant_guesty_credentials`
- Token refresh handled automatically by background job (tokens expire every 24 hours)

```
OAUTH2 FLOW:

  Customer Browser          Our Backend            Guesty Auth
  ----------------          -----------            -----------
  Click "Connect"  ------>  Generate auth URL
                            (client_id, redirect,
                             scopes, state)
                   <------  302 Redirect to Guesty
  Authorize App    ------------------------------------------>
                   <------------------------------------------
                            Auth code callback
  Redirect back    ------>  Exchange code for tokens -------->
                            Store encrypted tokens   <--------
                   <------  "Connected!" page
```

**Scopes requested:**
- `reservations:read` — pull bookings
- `listings:read` — pull property details
- `conversations:read` — pull past messages
- `conversations:write` — send replies
- `financials:read` — pull revenue data (Pro+ plans)

### Step 3: Property Discovery (5 minutes)

Automated background job triggers immediately after OAuth2 completes:

1. **Pull all listings** — GET /v1/listings (paginated, all properties)
2. **Pull active reservations** — GET /v1/reservations (status: confirmed, checked_in)
3. **Pull conversation history** — GET /v1/communication/conversations (last 90 days)
4. Store in `tenant_properties` and `messages` tables
5. Present property list to customer in dashboard: "We found X properties. Select which ones to activate."

### Step 4: Knowledge Base Build (10 minutes)

For each activated property, index the following into the tenant's vector DB namespace:

| Data Source | What Gets Indexed | Embedding Model |
|-------------|-------------------|-----------------|
| Property descriptions | Title, summary, amenities, house rules | text-embedding-3-small |
| Past host replies | Every outbound message from last 90 days | text-embedding-3-small |
| Guest FAQ patterns | Cluster common inbound questions | text-embedding-3-small |
| Check-in instructions | Parsed from listing or manually added | text-embedding-3-small |
| Local recommendations | Parsed from listing description | text-embedding-3-small |

- Each document tagged with: `tenant_id`, `property_id`, `doc_type`, `created_at`
- Initial index build: ~500-2000 documents per property (varies by message history)
- Processing time: ~10 minutes for a typical host with 5-10 properties

### Step 5: Agent Goes Live (1 minute)

1. Register webhook with Guesty for the tenant's account
2. Webhook URL: `https://api.aiguestagent.com/webhook/guesty/{tenant_id}`
3. Set tenant status to `active`
4. First message arrives → AI agent responds using tenant's knowledge base
5. Dashboard shows "Your agent is live" with green status indicator

### Onboarding Failure Handling

| Failure | Recovery |
|---------|----------|
| OAuth2 denied | Show "Connect Guesty" button again with explanation |
| Token exchange fails | Retry 3x, then show error with support link |
| No listings found | Prompt customer to check Guesty account has properties |
| KB build times out | Background retry, notify customer when complete |
| Webhook registration fails | Retry with backoff, alert engineering if persistent |

---

## 3. Host-Reply-Learner (The Moat)

**Build Estimate:** 2-3 weeks

This is the single most defensible feature. Every other AI guest messaging tool sends generic replies. Ours learns each host's unique style, tone, and knowledge — and gets better over time.

### Why This Is the Moat

```
  COMPETITOR APPROACH              OUR APPROACH
  --------------------             -------------------
  Generic prompt +                 Host's actual past replies +
  property description  =          property knowledge +
  "Hello! Thank you                host's tone/style =
   for your message..."            Reply indistinguishable
                                   from the host themselves
```

A competitor can copy our features. They cannot copy 6 months of a host's indexed reply history and progressive corrections. The longer a customer uses us, the harder it is to leave.

### Implementation: RAG over Fine-Tuning

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| Fine-tuning | Most accurate style match | Expensive per tenant ($50-500/tune), slow to update, model management nightmare at scale | NOT for MVP |
| RAG | Fast to deploy, instant updates, no per-tenant model costs, works with any LLM | Slightly less "native" style match | YES for MVP |
| Hybrid (future) | Best of both | Complex | Phase 3+ |

**MVP Decision: RAG.** We use retrieval-augmented generation with per-tenant vector namespaces.

### Retrieval Pipeline

```
  GUEST MESSAGE ARRIVES
          |
          v
  +-------------------+
  | Embed guest       |
  | message            |
  | (text-embedding-  |
  |  3-small)         |
  +-------------------+
          |
          v
  +-------------------+
  | Vector search in  |
  | tenant's namespace|
  |                   |
  | Query:            |
  |  - Similar past   |
  |    host replies   |
  |    (top 5)        |
  |  - Property docs  |
  |    (top 3)        |
  |  - FAQ matches    |
  |    (top 2)        |
  +-------------------+
          |
          v
  +-------------------+
  | Build prompt:     |
  |                   |
  | SYSTEM: You are   |
  | a host assistant. |
  | Match this style: |
  | [past replies]    |
  |                   |
  | CONTEXT:          |
  | [property docs]   |
  | [FAQ matches]     |
  |                   |
  | GUEST MESSAGE:    |
  | [the message]     |
  +-------------------+
          |
          v
  +-------------------+
  | LLM generates     |
  | reply in host's   |
  | style             |
  +-------------------+
          |
          v
  +-------------------+
  | Send via Guesty   |
  | API using         |
  | tenant's creds    |
  +-------------------+
```

### Progressive Learning Loop

This is where the moat deepens over time:

```
  AI SENDS REPLY
        |
        +---> Guest satisfied? ---> Reply indexed as "good example"
        |                           (added to vector DB)
        |
        +---> Host edits the AI draft?
                    |
                    v
              +------------------+
              | CORRECTION PAIR  |
              | stored:          |
              |                  |
              | original_ai:     |
              |   "The wifi      |
              |    password is   |
              |    on the        |
              |    fridge."      |
              |                  |
              | host_corrected:  |
              |   "Hey! WiFi    |
              |    pass is       |
              |    TinyHome2024  |
              |    - it's on     |
              |    the fridge    |
              |    too :)"       |
              +------------------+
                    |
                    v
              Host's correction
              gets HIGHER weight
              in future retrievals
```

**Key behaviors:**
- Every unedited AI reply gets indexed as a positive example (similarity score boost)
- Every host correction creates a correction pair — the corrected version gets indexed with higher weight
- Over time, the retrieval set becomes dominated by host-approved or host-written replies
- Result: after 2-4 weeks of usage, the AI's replies become nearly indistinguishable from the host's own writing

### Vector Database Architecture

```
VECTOR DB (pgvector on Supabase)

  Namespace: tenant_{uuid}
  +------------------------------------------+
  |  Collection: host_replies                |
  |    - embedding (1536 dimensions)         |
  |    - content (the reply text)            |
  |    - property_id                         |
  |    - guest_message (what prompted it)    |
  |    - source (host_original | ai_approved |
  |              | host_corrected)           |
  |    - weight (1.0 default, 1.5 for       |
  |              corrections, 0.8 for old)   |
  |    - created_at                          |
  +------------------------------------------+
  |  Collection: property_docs               |
  |    - embedding                           |
  |    - content                             |
  |    - property_id                         |
  |    - doc_type (rules | amenities |       |
  |                checkin | local_recs)     |
  +------------------------------------------+
  |  Collection: faq_patterns                |
  |    - embedding                           |
  |    - question_pattern                    |
  |    - best_answer                         |
  |    - property_id                         |
  |    - frequency (how often asked)         |
  +------------------------------------------+
```

### Isolation Guarantee

- Each tenant's vectors are stored in a separate namespace (partition key: `tenant_id`)
- All vector queries include `tenant_id` as a mandatory filter — no cross-tenant data leakage possible
- Deletion: when a tenant cancels, their entire namespace is purged

---

## 4. Billing — Stripe Integration

**Build Estimate:** 1-2 weeks

### Pricing Model

| Plan | Price | Includes | Target Customer |
|------|-------|----------|-----------------|
| **Trial** | Free (14 days) | Up to 3 properties, no credit card | Everyone |
| **Starter** | $49/property/mo | AI replies, basic dashboard, email support | 1-3 property hosts |
| **Pro** | $99/property/mo | + host-reply-learner, analytics, priority support | 4-20 property managers |
| **Business** | $199/property/mo | + API access, custom rules, phone support, SLA | 20+ property companies |

**Volume discounts:**
- 10+ properties: 15% off
- 25+ properties: 25% off
- 50+ properties: Custom pricing (contact sales)

### Stripe Integration Architecture

```
  SIGNUP                TRIAL                  CONVERT               RECURRING
+--------+          +----------+           +-----------+          +-----------+
| Clerk  |  ------> | 14-day   |  -------> | Stripe    |  ------> | Stripe    |
| Auth   |          | free     |           | Checkout  |          | Subscript.|
|        |          | trial    |           | Session   |          | (monthly) |
+--------+          +----------+           +-----------+          +-----------+
                         |                       |                      |
                    No credit card          Card captured          Webhook events
                    required                Subscription           (paid, failed,
                                            created                cancelled)
```

### Implementation Details

**Stripe Checkout:**
- Used for initial conversion from trial to paid
- Pre-built hosted page (no custom payment form needed for MVP)
- Supports cards, Apple Pay, Google Pay out of the box

**Stripe Subscriptions:**
- One subscription per tenant
- Metered billing based on active properties
- At the start of each billing period, count tenant's active properties
- Subscription item quantity = number of active properties x plan price

**Stripe Webhooks we handle:**

| Event | Action |
|-------|--------|
| `checkout.session.completed` | Activate paid plan, store stripe_customer_id |
| `invoice.paid` | Log payment, keep tenant active |
| `invoice.payment_failed` | Send email warning, 3-day grace period |
| `customer.subscription.deleted` | Suspend tenant, stop AI replies, retain data 30 days |
| `customer.subscription.updated` | Update plan tier in database |

**Trial-to-Paid Flow:**
1. Day 1: Signup, connect Guesty, agent goes live (no card)
2. Day 10: Email reminder — "Your trial ends in 4 days"
3. Day 13: Email reminder — "Last day tomorrow"
4. Day 14: Trial expires. Agent pauses. Dashboard shows "Upgrade to continue"
5. Customer clicks upgrade → Stripe Checkout → Agent resumes immediately

**Metered Property Counting:**
- Cron job runs on the 1st of each month
- Counts `tenant_properties WHERE active = true` for each tenant
- Updates Stripe subscription quantity via API
- Customer only pays for active properties

---

## 5. Customer Dashboard

**Build Estimate:** 3-4 weeks for MVP

### Tech Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Frontend | Next.js 14 (App Router) | SSR, fast, large ecosystem, Vercel deploy |
| Auth | Clerk | Pre-built components, Guesty OAuth compatible |
| API | Next.js API routes | Co-located with frontend, simpler deployment |
| Charts | Recharts | Lightweight, React-native, MIT licensed |
| UI Kit | shadcn/ui + Tailwind | Fast to build, professional look, no vendor lock |
| State | React Query (TanStack) | Server state management, caching, refetch |

### Dashboard Pages

```
NAVIGATION STRUCTURE:

/dashboard
  |-- Overview (default)
  |     Messages today, response time, automation rate
  |
  |-- /messages
  |     Full message log (sent/received)
  |     Filter by property, date, AI vs manual
  |     Click to expand conversation thread
  |
  |-- /properties
  |     List of connected properties
  |     Toggle active/inactive per property
  |     KB status (documents indexed, last updated)
  |
  |-- /analytics
  |     Response time trends (line chart)
  |     Messages per day (bar chart)
  |     Automation rate over time (area chart)
  |     Escalation log
  |
  |-- /settings
  |     Guesty connection status (reconnect button)
  |     Agent behavior (tone, escalation rules)
  |     Team members (invite, roles)
  |
  |-- /billing
  |     Current plan, active properties, next invoice
  |     Upgrade/downgrade, payment method, invoices
  |     Stripe Customer Portal link
```

### Overview Dashboard Wireframe

```
+------------------------------------------------------------------+
|  AI Guest Agent                          [Settings] [Logout]      |
+------------------------------------------------------------------+
|                                                                    |
|  +------------------+  +------------------+  +------------------+ |
|  | MESSAGES TODAY   |  | AVG RESPONSE     |  | AUTOMATION       | |
|  |       47         |  |     1m 23s       |  |      94%         | |
|  | +12 vs yesterday |  | -8s vs last week |  | +2% vs last wk  | |
|  +------------------+  +------------------+  +------------------+ |
|                                                                    |
|  +----------------------------------------------+  +------------+ |
|  | MESSAGES THIS WEEK                           |  | RECENT     | |
|  |                                              |  | ESCALATION | |
|  |  Mon  |||||||||||||||  42                    |  |            | |
|  |  Tue  |||||||||||||||||  51                  |  | Guest asked| |
|  |  Wed  ||||||||||||  38                       |  | about pet  | |
|  |  Thu  |||||||||||||||||||||  63  <-- today   |  | policy     | |
|  |  Fri  (projected)                            |  | 2:15 PM    | |
|  |                                              |  |            | |
|  +----------------------------------------------+  | Refund     | |
|                                                     | request    | |
|  +----------------------------------------------+  | 11:02 AM   | |
|  | RESPONSE TIME TREND (last 30 days)           |  |            | |
|  |    2m  .                                     |  +------------+ |
|  |        . .     .                             |                  |
|  |    1m    . . .   . . .  . .  .  .           |                  |
|  |              . .       .    .    . .         |                  |
|  |   30s                                 .     |                  |
|  |   _____________________________________|    |                  |
|  +----------------------------------------------+                  |
+------------------------------------------------------------------+
```

### Message Log View

| Column | Description |
|--------|-------------|
| Timestamp | When the message was sent/received (EST) |
| Property | Which property the conversation is about |
| Guest | Guest name |
| Direction | Inbound (guest) / Outbound (AI or host) |
| Content | Message preview (click to expand) |
| Response Time | Time between guest message and AI reply |
| Source | AI-generated, Host-edited, or Manual |
| Status | Sent, Delivered, Failed |

### Escalation Rules (Configurable per Tenant)

Default escalation triggers (host gets notified, AI does not reply):
- Guest mentions legal action, lawsuit, or attorney
- Guest requests a refund over $X (configurable threshold)
- Guest reports safety issue (fire, flood, gas, break-in)
- Guest message contains extreme negative sentiment
- AI confidence score below threshold (configurable, default 0.6)
- Guest explicitly asks to speak to the host/owner/manager

---

## 6. Infrastructure & Scaling

**Build Estimate:** 2-3 weeks (cloud migration)

### Current State (DLJ Internal)

```
  CURRENT ARCHITECTURE (7 properties, ~50 messages/day)

  +-------------------+         +-------------------+
  | Mac Studio        |         | Mac Mini          |
  | M2 Ultra 192GB   |         | M4 Pro 48GB      |
  |                   |         |                   |
  | Ollama            |         | Ollama            |
  | - Qwen 122B      |         | - Qwen 35B       |
  |   (primary)      |         |   (fallback)      |
  |                   |         |                   |
  | Webhook Server    |         |                   |
  | (Node.js)        |         |                   |
  +-------------------+         +-------------------+
          |
          v
  +-------------------+
  | Guesty API        |
  | (7 properties)    |
  +-------------------+
```

**This works great for DLJ.** We keep it running for our own properties. The SaaS product runs on separate infrastructure.

### Scaling Phases

#### Phase A: 10 Beta Customers (~50-100 properties, ~200-500 messages/day)

```
  +-------------------+     +-------------------+
  | Vercel            |     | Railway / Fly.io  |
  | (Next.js frontend)|     | (API + webhook    |
  |                   |     |  server)          |
  +-------------------+     +-------------------+
                                    |
                            +-------+-------+
                            |               |
                    +-------v----+  +-------v----+
                    | Supabase   |  | Redis      |
                    | (Postgres  |  | (BullMQ    |
                    |  + pgvector|  |  queues)   |
                    |  + Auth)   |  |            |
                    +------------+  +------------+
                            |
                    +-------v-------+
                    | Claude API    |
                    | (Haiku for    |
                    |  speed,       |
                    |  Sonnet for   |
                    |  quality)     |
                    +---------------+
                            |
                    +-------v-------+
                    | Guesty API    |
                    | (per-tenant   |
                    |  credentials) |
                    +---------------+
```

**Cost estimate at 10 customers:**
- Vercel Pro: $20/mo
- Railway: $20/mo
- Supabase Pro: $25/mo
- Redis (Upstash): $10/mo
- Claude API: ~$50-150/mo (500 msgs/day x $0.01-0.03/msg)
- **Total infrastructure: ~$125-225/mo**
- **Revenue at 10 customers x 5 properties avg x $99/mo: ~$4,950/mo**
- **Margin: ~95%**

#### Phase B: 50 Customers (~250-500 properties, ~1,000-2,500 messages/day)

Same architecture, scale vertically:
- Railway: upgrade to performance tier ($50/mo)
- Supabase: upgrade plan or move to dedicated Postgres ($100/mo)
- Redis: upgrade tier ($30/mo)
- Claude API: ~$250-750/mo
- **Total infrastructure: ~$450-930/mo**
- **Revenue: ~$24,750/mo**
- **Margin: ~96%**

#### Phase C: 200+ Customers (~1,000-2,000 properties, ~5,000-10,000 messages/day)

Consider:
- Move to AWS/GCP with dedicated infrastructure
- Kubernetes for orchestration
- Dedicated PostgreSQL (RDS or Cloud SQL)
- Self-hosted Redis cluster
- Possibly run local LLMs on GPU instances for cost optimization at this volume

### LLM Strategy Decision

| Approach | Cost/Message | Latency | Quality | Scalability | Verdict |
|----------|-------------|---------|---------|-------------|---------|
| Local Ollama (current) | ~$0 | 3-8s | Good (122B) | Poor (hardware bound) | Keep for DLJ only |
| Claude Haiku | ~$0.005 | 1-2s | Good | Excellent | DEFAULT for MVP |
| Claude Sonnet | ~$0.03 | 2-4s | Excellent | Excellent | Pro/Business plans |
| GPT-4o-mini | ~$0.003 | 1-2s | Good | Excellent | Backup provider |

**Recommendation:** Use **Claude Haiku** as the default model for all customer-facing replies. Offer **Claude Sonnet** as an upgrade for Pro/Business plans. Keep **GPT-4o-mini** as a failover if Anthropic API has an outage. This gives us provider redundancy without operational complexity.

### Message Processing SLA Targets

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| Webhook-to-reply latency | < 90 seconds | > 120 seconds |
| Queue processing time | < 10 seconds | > 30 seconds |
| LLM inference time | < 15 seconds | > 30 seconds |
| Guesty API send time | < 5 seconds | > 10 seconds |
| End-to-end uptime | 99.5% | Any downtime |

---

## 7. Security

**Build Estimate:** 1 week (integrated into other components)

### Credential Security

```
  GUESTY TOKEN LIFECYCLE

  Customer authorizes via OAuth2
          |
          v
  +-------------------+
  | Tokens received   |
  | (access + refresh)|
  +-------------------+
          |
          v
  +-------------------+
  | Encrypt with      |
  | AES-256-GCM       |
  | (unique IV per    |
  |  encryption)      |
  +-------------------+
          |
          v
  +-------------------+
  | Store in          |
  | tenant_guesty_    |
  | credentials table |
  | (encrypted blob   |
  |  + IV, never      |
  |  plaintext)       |
  +-------------------+
          |
          v
  +-------------------+
  | Encryption key    |
  | stored in env var |
  | (never in DB,     |
  |  never in code)   |
  +-------------------+
```

**Key management:**
- Master encryption key stored as environment variable on hosting platform (Railway/Fly.io secrets)
- Never committed to source control
- Rotated quarterly (re-encrypt all stored credentials)
- In production (Phase C+): migrate to AWS KMS or HashiCorp Vault

### Tenant Isolation

| Layer | Isolation Method |
|-------|-----------------|
| Database | Row-Level Security (RLS) in PostgreSQL. Every table with tenant data has `tenant_id` column. RLS policies enforce that queries only return rows matching the authenticated tenant. |
| Vector DB | Namespace isolation. Each tenant's embeddings stored in `tenant_{uuid}` namespace. All vector queries require namespace parameter. |
| Message Queue | Named queues per tenant. Workers only process messages from assigned queue. |
| API | JWT tokens include `tenant_id` claim. Middleware validates tenant_id on every request. |
| Logs | Tenant IDs included in structured logs but credential values NEVER logged. |

### What We Never Do

- Never log Guesty API tokens (not even masked)
- Never expose credentials in API responses
- Never store credentials in plaintext
- Never include customer data in error reports sent to third parties
- Never share a tenant's knowledge base data with another tenant
- Never train shared models on customer data without explicit consent

### Network Security

| Measure | Implementation |
|---------|---------------|
| TLS | HTTPS everywhere. TLS 1.3. HSTS headers. |
| Rate limiting | Per-tenant: 100 req/min API, 500 req/min webhooks. Global: 10K req/min. |
| API auth | Dashboard API: JWT (via Clerk). Webhook ingress: HMAC signature verification. |
| CORS | Dashboard origin only. No wildcard. |
| CSP | Content Security Policy headers on all pages. |
| Input validation | Zod schemas on all API inputs. Sanitize all user-provided text. |

### Data Retention & Deletion

| Action | Behavior |
|--------|----------|
| Customer requests data export | Generate JSON export of all their data within 48 hours |
| Customer cancels subscription | Data retained 30 days, then purged (messages, properties, credentials, vectors) |
| Customer requests immediate deletion | All data purged within 24 hours, confirmation email sent |
| Message data | Retained for duration of subscription + 30-day grace period |
| Guesty credentials | Deleted immediately on cancellation (no grace period) |

### SOC 2 Roadmap (Not MVP)

For Phase C (200+ customers), we will need SOC 2 Type II:
- Formal access control policies
- Annual penetration testing
- Incident response plan
- Vendor risk assessment
- Audit logging (already built into MVP via structured logs)
- Target: begin SOC 2 process at ~100 paying customers

---

## 8. Summary & Roadmap

### Total MVP Build Estimate

| Component | Estimated Build Time | Dependencies |
|-----------|---------------------|--------------|
| Multi-tenant backend + tenant routing | 3-4 weeks | None (start here) |
| Onboarding flow + Guesty OAuth2 | 3-4 weeks | Multi-tenant backend |
| Host-reply-learner (RAG pipeline) | 2-3 weeks | Onboarding flow |
| Stripe billing integration | 1-2 weeks | Multi-tenant backend |
| Customer dashboard | 3-4 weeks | All backend components |
| Security hardening | 1 week | Integrated throughout |
| Cloud infrastructure setup | 2-3 weeks | Parallel with development |
| **Total** | **10-14 weeks** | |

### Recommended MVP Scope (First Beta Customer)

Strip to the essentials needed to onboard one paying customer:

1. Onboarding flow (signup + Guesty OAuth2 + property discovery)
2. Tenant isolation (multi-tenant message routing)
3. Host-reply-learner (RAG pipeline — the moat)
4. Basic dashboard (message log + property list)
5. Stripe billing (trial + subscription)

**Reduced MVP estimate: 6-8 weeks**

### Phased Roadmap

```
PHASE 1: FOUNDATION (Weeks 1-4)
+---------------------------------------------------------------+
| Week 1-2: Multi-tenant backend                                |
|   - PostgreSQL schema + RLS policies                          |
|   - Tenant CRUD APIs                                          |
|   - Webhook ingress + tenant routing                          |
|   - BullMQ message queues                                     |
|   - Cloud infrastructure setup (Supabase, Railway, Vercel)    |
|                                                                |
| Week 3-4: Onboarding + Guesty OAuth2                          |
|   - Clerk auth integration                                    |
|   - Guesty OAuth2 flow (authorize, token exchange, storage)   |
|   - Property discovery (auto-pull listings)                    |
|   - Webhook registration per tenant                            |
|   - End-to-end test: signup to first AI reply                 |
+---------------------------------------------------------------+

PHASE 2: MOAT + MONETIZATION (Weeks 5-8)
+---------------------------------------------------------------+
| Week 5-6: Host-Reply-Learner                                  |
|   - pgvector setup + embedding pipeline                       |
|   - Index past host replies + property docs                   |
|   - RAG retrieval pipeline (embed → search → prompt)          |
|   - Progressive learning (correction indexing)                |
|   - Per-tenant namespace isolation                             |
|                                                                |
| Week 7-8: Dashboard + Billing                                 |
|   - Next.js dashboard (message log, properties, overview)     |
|   - Stripe Checkout + Subscriptions                           |
|   - Trial-to-paid conversion flow                             |
|   - Basic analytics (response times, message volume)          |
|   ** FIRST BETA CUSTOMER TARGET: END OF WEEK 8 **            |
+---------------------------------------------------------------+

PHASE 3: POLISH + SCALE PREP (Weeks 9-12)
+---------------------------------------------------------------+
| Week 9-10: Dashboard v2                                       |
|   - Analytics charts (trends, automation rate)                |
|   - Escalation log + configuration                            |
|   - Agent behavior settings (tone, rules)                     |
|   - Team member invites                                       |
|                                                                |
| Week 11-12: Hardening                                         |
|   - Security audit (penetration testing)                      |
|   - Load testing (simulate 50 tenants)                        |
|   - Monitoring + alerting (Datadog or Grafana)                |
|   - Documentation (API docs, onboarding guide)                |
|   - Error handling + retry logic polish                       |
|   ** 10 BETA CUSTOMERS TARGET: END OF WEEK 12 **             |
+---------------------------------------------------------------+

PHASE 4: GROWTH (Weeks 13+)
+---------------------------------------------------------------+
| - Marketing launch (blog, ProductHunt, HN)                    |
| - Self-serve onboarding (no manual steps)                     |
| - Fine-tuning option for Business plan                        |
| - Multi-PMS support (Hostaway, Lodgify, OwnerRez)            |
| - White-label option for property management companies        |
| - Mobile app (React Native)                                   |
| - SOC 2 certification process                                 |
+---------------------------------------------------------------+
```

### Key Metrics to Track from Day 1

| Metric | Target | Why It Matters |
|--------|--------|---------------|
| Signup-to-live time | < 30 minutes | Activation friction kills SaaS |
| Message response latency | < 90 seconds | Must beat human response time |
| Automation rate | > 85% | Hosts won't pay if they still answer most messages |
| Host edit rate | < 20% (decreasing over time) | Proves the learner works |
| Trial-to-paid conversion | > 15% | Revenue viability |
| Monthly churn | < 5% | Sustainability |
| Net Promoter Score | > 50 | Word-of-mouth growth |

### Revenue Projections

| Milestone | Customers | Avg Properties | Avg Plan | MRR | ARR |
|-----------|-----------|---------------|----------|-----|-----|
| Week 8 | 1 (beta) | 5 | $99 | $495 | $5,940 |
| Week 12 | 10 (beta) | 7 | $99 | $6,930 | $83,160 |
| Month 6 | 50 | 8 | $99 | $39,600 | $475,200 |
| Month 12 | 200 | 10 | $99 | $198,000 | $2,376,000 |
| Month 24 | 1,000 | 12 | $99 | $1,188,000 | $14,256,000 |

### Risk Register

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| Guesty changes API / revokes access | Critical | Low | Maintain relationship, diversify to other PMS platforms |
| Claude API price increase | Medium | Medium | Multi-provider support (GPT-4o-mini fallback) |
| Competitor launches similar product | Medium | High | Host-reply-learner moat + first-mover + speed of execution |
| Data breach | Critical | Low | Encryption at rest, RLS, security audit, no credential logging |
| LLM generates harmful/incorrect reply | High | Medium | Confidence scoring, escalation rules, host edit loop |

---

## Appendix A: Technology Stack Summary

| Layer | Technology | Alternative |
|-------|-----------|-------------|
| Frontend | Next.js 14, Tailwind, shadcn/ui | Remix |
| Auth | Clerk | Auth0 |
| Backend API | Next.js API Routes (Node.js) | Fastify standalone |
| Database | Supabase (PostgreSQL 15) | PlanetScale, Neon |
| Vector DB | pgvector (via Supabase) | Pinecone, Weaviate |
| Queue | BullMQ (Redis) | AWS SQS |
| Redis | Upstash | Redis Cloud |
| LLM (default) | Claude Haiku | GPT-4o-mini |
| LLM (premium) | Claude Sonnet | GPT-4o |
| Embeddings | text-embedding-3-small (OpenAI) | Cohere embed-v3 |
| Payments | Stripe | None (Stripe is the standard) |
| Hosting (FE) | Vercel | Netlify |
| Hosting (BE) | Railway | Fly.io, Render |
| Monitoring | Datadog | Grafana + Loki |
| Error tracking | Sentry | Bugsnag |
| Email | Resend | Postmark |

## Appendix B: API Endpoints (MVP)

```
AUTH
  POST   /api/auth/signup          Clerk-managed
  POST   /api/auth/login           Clerk-managed

ONBOARDING
  POST   /api/onboarding/guesty    Start OAuth2 flow
  GET    /api/onboarding/callback   OAuth2 callback
  POST   /api/onboarding/discover   Trigger property discovery
  POST   /api/onboarding/activate   Activate selected properties

PROPERTIES
  GET    /api/properties            List tenant's properties
  PATCH  /api/properties/:id        Update property (active/inactive)
  GET    /api/properties/:id/kb     Get KB status for property

MESSAGES
  GET    /api/messages              List messages (paginated, filterable)
  GET    /api/messages/:id          Get single message + thread

ANALYTICS
  GET    /api/analytics/overview    Dashboard overview stats
  GET    /api/analytics/response    Response time trends
  GET    /api/analytics/volume      Message volume over time

BILLING
  POST   /api/billing/checkout      Create Stripe Checkout session
  GET    /api/billing/subscription  Get current subscription
  POST   /api/billing/portal        Create Stripe Customer Portal session
  POST   /api/webhooks/stripe       Stripe webhook handler

WEBHOOK (PUBLIC)
  POST   /webhook/guesty/:tenant_id  Guesty webhook ingress
```

---

**Document prepared by the CTO, DLJ Properties.**
**For internal use. Do not distribute without CEO approval.**
