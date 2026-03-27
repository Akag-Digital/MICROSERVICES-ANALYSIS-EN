# Technical Analysis: Technical Debt, Scale Bottlenecks, and the Case for Microservices

## 0. Context: Why the Refactor is Necessary

The system was originally designed to follow the **same container logic as Rove LAB** — a simplified approach, built to operate with **at most 1,000 SKUs**. In that context, a Remix monolith was sufficient and appropriate for the existing complexity.

Over time, new features and business rules were added — especially **parent-child bundle synchronization, ETA validation, bulk import/export, and webhook-based order processing** — without revisiting the base architecture. Each new feature was plugged into the existing structure, which was never designed to support them at scale.

The result is that the **impact of these changes hit the system at its foundation**: synchronization logic, which should be an isolated and asynchronous process, now runs in the same process that serves the UI and responds to webhooks. What was a solid base for 1,000 SKUs becomes a structural bottleneck for 12,000.

This is not a matter of poor implementation — the system served its intended purpose well. It is a matter of **scope evolution**: the new business rules require a different architecture, and this refactor is the natural response to that growth.

### The Prisma Schema Did Not Keep Up with the Evolution

The database structure reflects the original simplified logic and has accumulated limitations that now directly impact the synchronization flow and scalability:

**1. `ETA`, `compressionDate`, and `completedAt` are `String` in the `Container` model**
```prisma
model Container {
  ETA             String   // should be DateTime
  compressionDate String?  // should be DateTime?
  completedAt     String?  // should be DateTime?
}
```
Date comparisons happen in application code, not in the database. It is impossible to run `WHERE ETA < NOW()` with an index — the ETA validation cron must load all containers into memory and compare in JS.

**2. `Variant` and `ProductsWithoutContainers` are parallel models with duplicated fields**
```prisma
// Variant has:                     preOrderStock, preOrderSales, status, startDate, endDate, templateId, stockManagement...
// ProductsWithoutContainers has:   preOrderStock, preOrderSales, status, startDate, endDate, templateId, stockManagement...
```
These are two separate tables representing similar concepts with no shared base. All synchronization logic must be written twice — once per model — and any new pre-order feature must be implemented in both.

**3. `ProductChildren` uses Shopify strings as references instead of FKs**
```prisma
model ProductChildren {
  shopifyChildId  String?  // loose string, no FK to Variant
  shopifyParentId String?  // loose string, no FK to ProductsWithoutContainers
  parentId        Int?     // FK to ProductsWithoutContainers (nullable)
  childId         Int?     // field with no FK defined
}
```
The relationship between a child and its actual `Variant` record is resolved by string matching in code, not by referential integrity. This means a stale `shopifyChildId` is never detected by the database — it becomes silently inconsistent data.

**4. `Variant.containerId` is required (non-null)**
```prisma
model Variant {
  containerId Int  // required — every variant MUST have a container
}
```
In the bundle flow, children of `ProductsWithoutContainers` exist without a direct container. This forces workarounds in code to create artificial containers or maintain two completely separate code paths for the same concept of "pre-order variant".

**5. `preOrderStock` and `preOrderSales` as direct row fields**
```prisma
preOrderStock Int @default(0)
preOrderSales Int @default(0)
```
Multiple order webhooks processed in parallel each run `UPDATE variant SET preOrderSales = preOrderSales + qty` on the same row simultaneously. Without explicit concurrency control, this is a race condition at scale — two webhooks processing at the same time can overwrite each other.

**6. No `shop` field on `Variant`, `Container`, or `OrderItem`**
```prisma
model Variant   { /* no shop */ }
model Container { /* no shop */ }
model OrderItem { /* no shop */ }
```
To filter data by shop, a join is required: `Variant → Product → shop`. In synchronization queries with 12k products, this chained join significantly increases query cost and prevents simple shop-based indexes.

**7. `WebhookIdempotency` and `Webhooks` are separate overlapping tables**
```prisma
model Webhooks           { orderId BigInt @unique; data Json? }  // payload
model WebhookIdempotency { orderId BigInt; webhookType String }  // deduplication
```
Idempotency control grew as a separate table instead of being consolidated. With async queues (required for 12k products), this logic would need to be rethought anyway.

---

These schema issues do not break current functionality, but they **multiply the cost of every synchronization operation** and make any extraction into microservices more complex than it needs to be. Schema redesign is part of the architectural refactor.

---

## 1. Current State: Remix Monolith Wearing Multiple Hats

The project is a **Shopify Admin App** running everything in a single Node.js/Remix process:
- Serves the admin UI (React/Polaris)
- Processes webhooks in real time
- Executes blocking CSV imports/exports
- Makes synchronous calls to the Shopify API
- Runs external cron job scripts using the same DB

This works for hundreds of products. For **12,000 products**, each of these roles will collide with the others.

---

## 2. Technical Debt Identified

### 2.1 Dead Code and Severe Duplication

The `refactoring-plan.md` already catalogued ~1,155 eliminable lines, but the problem goes beyond cosmetics:

| Problem | File(s) | Impact |
|---|---|---|
| Two quasi-identical `orderTagManager` files | `.js` (~1,389 lines) + `.server.js` (~1,595 lines) | Duplicated maintenance, risk of diverging bugs |
| 8 selection hooks with identical code | `useAddProductsSelection`, `useAddProductChildrenSelection`, etc. | Any bug must be fixed 8 times |
| 4 implementations of `searchProducts` | `products-sku`, `products-by-container`, `products`, `products-parents` | Behaviors can silently diverge |
| `normalizeToVariantGid()` in 4 different places | `orderTagManager`, `separateItemsByContainer`, `syncParentShopifyInventoryNoPreorder`, `csvUpload` | Each may normalize differently, causing ID mismatches |
| `escapeCSV`, `ensureExportsDir`, `streamFinish`, `getUniqueExportFileName` duplicated | `containers/csvExport.js` and `products-parents/csvExport.js` | Any CSV improvement must be made in both places |

### 2.2 Webhooks Registered Manually (Architectural Decision)

The critical webhooks are **active in production**, registered **manually** via Shopify API — intentionally outside `shopify.app.toml`. This is an architectural decision to maintain full control over when and how webhooks are registered, independent of the app's deploy cycle.

```
orders/create           ← active (manual registration)
orders/updated          ← active (manual registration)
orders/cancelled        ← active (manual registration)
inventory_levels/update ← active (manual registration)
products/update         ← active (manual registration)
```

**This pattern must be maintained.** Manual registration ensures that changes to `shopify.app.toml` or `npm run deploy` executions do not interfere with production webhooks.

### 2.3 Monolithic Files without Separation of Concerns

| File | Size | Problem |
|---|---|---|
| `orderTagManager.server.js` | ~1,595 lines / 48KB | Tag generation, order processing, and inventory logic all mixed together |
| `containerActions.js` | 857 lines | Create, update, archive, and calculate container stock in one file |
| `products-container-id.action.server.js` | 812 lines | Multiple form `intent` handlers in one giant function |
| `parentPreOrderAndContainers.js` | 766 lines | Status analysis + container fetching + inventory calculation |
| `ContainersTable.jsx` | 833 lines | UI component with complex business logic embedded |

### 2.4 CSV Export to Local Disk

```javascript
// app/utils/containers/csvExport.js
// Writes file to: public/exports/export_container_data.csv
```

CSV files are written to `public/exports/` on the server's local disk. In a multi-replica environment (required for 12k products), **each replica would have its own files**, and the replica that served the download may not be the one that generated the CSV.

### 2.5 Three Error Handling Patterns Coexisting

```javascript
// Pattern A — correct (with logger)
} catch (error) { logError("context", error); throw error; }

// Pattern B — silences context
} catch (error) { throw error; }

// Pattern C — no try/catch (error bubbles uncaught)
const result = await riskyOperation();
```

In production with 12k products, an unlogged error in a webhook can cause Shopify to retry the same webhook hundreds of times without any visibility into what went wrong.

---

## 3. Why 12,000 Products Will Break — Concrete Scale Bottlenecks

### 3.1 ETA Validation: Serial Loop per Product

```javascript
// app/utils/etaValidationManager.js — line 30
for (const parentProduct of parentProducts) {            // loop over ALL parents
  const childVariants = await prisma.variant.findMany({  // 1 query per parent
    where: { shopifyId: { in: childShopifyIds } }
  });
  // ... more queries, more Shopify API calls
}
```

**The problem:** The ETA validation cron makes **at least 1 database query per parent product**. With 12,000 products (hypothetical: 3,000 parents × 4 children each), that is **3,000 sequential queries** just to load the data — not counting the Shopify API calls that follow to update policies.

The cron will hold the database the entire time, making the UI slow for all users.

### 3.2 CSV Import: Query per Line

```javascript
// app/utils/containers/csvUpload.js
async function getOrCreateTemplate(templateName, shop) {
  let template = await prisma.template.findFirst({ ... }); // query per CSV line
  if (!template) {
    template = await prisma.template.create({ ... });       // create per CSV line
  }
  return template;
}
```

A CSV with 12,000 products can trigger 12,000+ `getOrCreateTemplate` calls, all synchronous in the main Remix process, **blocking all other UI requests** during that time.

Additionally, the Products Parents import makes Shopify GraphQL calls in chunks of 100 variants:

```javascript
// app/utils/products-parents/csvUpload.js
const CHUNK = 100;
for (let i = 0; i < unique.length; i += CHUNK) {
  const response = await admin.graphql(query, { variables: { ids: chunk } });
}
```

12,000 products = 120 sequential GraphQL calls, during which the Remix process is completely blocked.

### 3.3 Bundle Sync: N+1 Queries

```javascript
// app/utils/updateChildrenInContainer.js
for (const item of itemsWithContainers) {     // loop per order item
  for (const child of item.productChildren) { // loop per child
    rawOps.push({ childId, containerId, qty });
  }
}
// Then: 1 findMany + N updates (still N transactions)
```

An order with 10 items × 5 children each = 50 DB operations inside the webhook handler. When the `orders/create` webhook is active, **Shopify expects a response within 5 seconds** before considering it a failure and retrying. If the database is slow (because the cron is running a 12k product loop), the webhook will fail and be retried, creating duplicates.

### 3.4 Public API without Cache

```javascript
// /api/get-product — called by theme JavaScript on EVERY pageview
// Calculates stockCalc = preOrderStock - preOrderSales in real time
// No cache, no Redis, no CDN
```

A store with 12,000 pre-order products and 1,000 simultaneous visitors = 1,000 concurrent MySQL queries, all going through the same Remix process that is also serving the admin UI and processing webhooks.

### 3.5 External Cron Jobs Share the Same Database

```bash
# package.json
npm run validate-eta:all       # scripts/cronjob-validate-eta.js
npm run sync:products-parents  # scripts/sync-products-parents.js
```

These scripts run outside the Remix process but use the **same Prisma client pointing to the same MySQL**. When running, they compete for connections with the UI and webhooks. With 12k products, the cron will hold the connection pool for an extended time.

---

## 4. Why Microservices Are the Right Solution

The issue is not just performance — it is **failure isolation** and **independent scalability**. Currently, a heavy CSV import freezes the UI. A slow cron saturates the public API. A webhook bug can take everything down.

### Proposed Architecture

```
                    ┌─────────────────────┐
                    │   Shopify Webhooks   │
                    └──────────┬──────────┘
                               │ HTTP (5s timeout)
                               ▼
              ┌────────────────────────────────┐
              │     Webhook Ingestor           │  ← receives and ACKs immediately
              │   (lightweight Node.js)        │
              └──────────────┬─────────────────┘
                             │ publishes to queue
                             ▼
              ┌─────────────────────────────────┐
              │          Message Queue          │
              │     (BullMQ + Redis  or  SQS)   │
              └──┬──────────┬──────────┬────────┘
                 │          │          │
     ┌───────────┘   ┌──────┘   ┌─────┘
     ▼               ▼          ▼
┌─────────┐  ┌──────────┐  ┌──────────────┐
│  Order  │  │Inventory │  │   Product    │
│ Worker  │  │  Worker  │  │   Worker     │
│(orders) │  │(stock)   │  │  (updates)   │
└────┬────┘  └────┬─────┘  └──────┬───────┘
     │             │               │
     └──────────┬──┘               │
                ▼                  ▼
     ┌─────────────────┐   ┌───────────────────┐
     │   MySQL (RDS)   │   │  Shopify Admin API │
     └─────────────────┘   └───────────────────┘

┌─────────────────────────────────────────────────────┐
│               Independent Services                  │
├────────────────┬────────────────┬───────────────────┤
│  CSV Service   │  Sync Service  │  Storefront API   │
│ (import/export)│  (ETA crons,   │  (get-product,    │
│  async with    │  bundle sync)  │  badges — cached) │
│  progress      │  scheduled     │  Redis/CDN        │
└────────────────┴────────────────┴───────────────────┘

┌─────────────────────────────────────────────────────┐
│              Remix App (unchanged)                  │
│  ONLY: admin UI + Shopify authentication            │
│  No heavy processing, no cron, no CSV sync          │
└─────────────────────────────────────────────────────┘
```

---

## 5. Each Microservice in Detail

### 5.1 Webhook Ingestor — Critical Isolation

**Current problem:** The webhook handler inside Remix makes DB queries and Shopify API calls within Shopify's 5s timeout. If it fails, it generates duplicate retries that `WebhookIdempotency` must filter out.

**Solution:** A separate service that only does **two things**: validate HMAC + publish to the queue. Responds 200 in <100ms. The actual processing happens in Workers with automatic retry, exponential backoff, and dead-letter queue.

**Suggested stack:** Node.js + Fastify (lightweight and fast) + BullMQ + Redis

**Why separate:**
- Can have zero downtime during Remix deploys
- Can scale horizontally without affecting the UI
- Worker failures do not delay the ACK to Shopify

---

### 5.2 CSV Service — Without Blocking the UI

**Current problem:** Uploading a 12k-line CSV blocks the Remix process, freezing all other users. Exports are saved to local disk (`public/exports/`), incompatible with multiple replicas.

**Solution:** CSV Service accepts the file, responds immediately with a `jobId`, processes in background with BullMQ. The UI polls the `jobId` for progress (0%... 45%... 100%). Exported files go to **Digital Ocean Spaces** (not local disk), with a pre-signed URL for direct download.

**Suggested stack:** Node.js Worker + BullMQ + Digital Ocean Spaces + progress events via SSE or polling

**Why separate:**
- Can process large files without affecting Remix latency
- Can have dedicated workers with more memory (`--max-old-space-size`)
- Parallel exports from multiple shops do not compete with each other
- Digital Ocean Spaces URL works on any replica, with no disk coupling

---

### 5.3 Sync Service — Cron with Real Control

**Current problem:** `scripts/cronjob-validate-eta.js` runs sequentially across all shops, holding database connections for an extended period. No queue, no retry, no observability. Competes with the UI and webhooks for the same MySQL connection pool.

**Solution:** Sync Service with a scheduler (BullMQ Scheduler or Cloud cron). Validates ETA processing shops in parallel with controlled concurrency (`concurrency: 5`). Each shop is an independent job — if one fails, the others continue. Uses its own MySQL connection pool with lower priority.

**Suggested stack:** BullMQ Scheduler + Redis + dedicated Prisma pool (separate connection limit)

**Why separate:**
- The cron today competes with webhooks and UI for the same MySQL connection pool
- Separated, it can have its own lower-priority pool without impacting UX
- Failed jobs go into a dead-letter queue for manual reprocessing
- Real observability: BullMQ dashboard shows active, failed, and average job times

---

### 5.4 Storefront API — Edge Cache

> Analyze the possibility of using an edge service (Cloudflare Workers) for public endpoints called by the Shopify theme, with a 60s Redis cache to reduce MySQL load — or another model.

**Current problem:** `/api/get-product` is called on every Shopify theme pageview, without cache, going through Remix to MySQL. Under high traffic, it saturates the database.

**Important context:** this endpoint **is not part of the Shopify Admin**. It is a public API, with no authentication, called by theme JavaScript in the end customer's browser — completely separate from the Remix app embedded in the admin. The admin app (Remix + Polaris) **does not change** and continues to be embedded normally in the Shopify Admin.

**Solution:** Extract only the public endpoints (`/api/get-product`, `/api/get-checkout-badges`, `/api/validate-eta`) to an edge service with a 60s Redis cache. The query is read-heavy and stateless — a perfect candidate for edge.

**Suggested stack:** Cloudflare Workers + Redis (Upstash) + cache-control headers

**Why separate:**
- ~10ms latency (edge cache) vs ~200ms (Remix → MySQL)
- Can handle 100,000 requests/min without impacting the admin
- Scales automatically on the CDN, no extra instance cost
- Cache can be selectively invalidated when a product changes (via webhook)
- Shopify already uses Cloudflare as CDN — the store domain already passes through their network, the Worker intercepts before leaving it

**Clear separation of responsibilities:**

| | Remix App (unchanged) | Storefront API (edge) |
|---|---|---|
| Who uses it | Merchant in Shopify Admin | End customer browsing the store |
| Authentication | Shopify OAuth required | None — public endpoint |
| Frequency | Low | Very high (every pageview) |
| Can go to edge? | No | Yes |

**Simpler alternative:** if you prefer to keep everything on DigitalOcean, a Redis with 60s TTL in front of the Remix endpoint resolves the MySQL load problem without new infrastructure — it does not solve geographic latency, but eliminates the main bottleneck.

---

### 5.5 Mutations API — Backend for Changes

**Current problem:** Product configuration changes (enabling pre-order, changing templates, updating stock) go through the Remix action, which calls the Shopify API **synchronously** while the user waits on screen.

**Solution:** A dedicated API service (Express or Fastify) that receives mutations, applies them to the DB, and **queues** the Shopify API updates. The UI receives immediate confirmation; propagation to the Shopify API happens in background with controlled rate limiting. The existing `shopifyAdminRestThrottle.js` — which already exists — is moved here.

**Suggested stack:** Fastify + BullMQ (Shopify mutation queue) + Redis (rate limit state)

**Why separate:**
- User does not wait for the Shopify API to respond
- Centralized rate limiting: multiple users cannot exceed the Shopify API ceiling
- Can retry failed mutations without impacting UX
- Remix becomes only a UI/auth layer, with no heavy business logic

---

## 6. Impact of Each Bottleneck with the Solution

| Scenario | Monolith (12k products) | With Microservices |
|---|---|---|
| ETA cron running | UI freezes, webhooks delayed | Isolated process, no UI impact |
| CSV import of 5k lines | All users unresponsive | Async job, UI stays free, real-time progress |
| Peak of 1k simultaneous store pageviews | Public API saturates MySQL, admin slows down | Edge cache absorbs, MySQL sees no load |
| Shopify fires 500 webhooks | Remix process may lose or duplicate events | Queue guarantees ordered processing with retry |
| New version deploy | In-flight webhooks killed | Ingestor stays up; Workers drained before stopping |
| Bug in CSV import | Brings down the main process | Only the CSV worker stops; UI and webhooks continue |
| Shopify API rate limit hit | 429 errors surface to the user | Queue holds, retries with backoff, user sees no error |

---

## 7. Recommended Extraction Order (Lowest Risk → Highest Gain)

```
Phase 1 — Without touching the Remix core
  1. Webhook Ingestor + BullMQ/Redis
     ↳ Extract handlers, put queue in front
     ↳ Remix keeps receiving, but delegates immediately

  2. Storefront API on Edge
     ↳ /api/get-product → Cloudflare Worker + Redis cache
     ↳ Removes heavy load from main MySQL

Phase 2 — Extract heavy processing
  3. Async CSV Service
     ↳ Accept upload, return jobId, process in worker
     ↳ Digital Ocean Spaces for exported files

  4. Sync Service for cron
     ↳ ETA validation + sync-products-parents as scheduled jobs
     ↳ Separate connection pool to MySQL

Phase 3 — Mutations backend
  5. Mutations API Service
     ↳ Write operations with queuing for Shopify API
     ↳ Remix becomes only a UI/auth layer

```

---

## 8. Final Note

The project already has a structure that **anticipates** this separation. The division between `endpoints/`, `utils/`, `hooks/`, and `components/` respects domain boundaries that map directly to the microservices. The work is primarily about **moving code that already exists** into processes that can scale and fail independently — not rewriting from scratch.

The greatest risk of not making this separation is not the isolated slowness of a single feature: it is the **cascade effect**. A slow CSV import freezes Remix, which delays webhooks, which cause Shopify to retry, which further overloads the database, which slows the cron, which leaves ETAs outdated, which causes overselling of already-exhausted pre-order stock.

---

## 9. Note on the Suggestions in This Document

The stacks, extraction phases, caching models, and architectural decisions described here are **initial suggestions based on the analysis of the current state** — not a definitive plan.

A **more detailed and direct planning session** will still be conducted, considering real infrastructure constraints, business priorities, and execution capacity. Decisions such as tool selection (BullMQ vs SQS, Cloudflare vs Redis on DO, schema redesign vs incremental adaptation) will be revisited at that stage.

This document serves as a diagnosis and starting point for that conversation — not as a final technical specification.
