# Payment Service System Design (with 3rd Party PSP)

# Important Concepts

- Payment state machine (idempotency, exactly-once semantics)
- Double-entry ledger / bookkeeping
- PSP integration patterns (hosted checkout vs. API integration)
- Reconciliation between internal records and PSP/bank statements
- PCI-DSS compliance and tokenization

# Context

Imagine we are building the payment system for an Amazon-scale e-commerce platform. Some real-world reference numbers:

- **Amazon** processes ~1.6M orders/day on average, but each order can generate 3-5 payment transactions (authorization, capture, marketplace seller splits, refunds, disputes). That's **~5-8M payment transactions/day**, and on **Prime Day / Black Friday, 5-10x** that (~50M+/day).
- **Adyen** (a major PSP) hit **160,000 transactions/minute** (~2,700 TPS) during Black Friday 2024.
- **Stripe** processed **$1.4 trillion** in total payment volume across 2024.

For this design, we target **Amazon marketplace scale**: a platform with millions of buyers, hundreds of thousands of sellers, and a payment system that must handle tens of millions of transactions per day with peak bursts in the thousands of TPS.

When a customer places an order, our system needs to:

1. Accept a payment intent from the order service.
2. Collect payment method details securely (via PSP-hosted UI / tokenization).
3. Charge the customer through the PSP.
4. Track the payment lifecycle (authorized → captured → settled, or failed/refunded).
5. Record every money movement in an internal ledger for accounting.
6. Handle refunds and disputes initiated by either the merchant or the customer.
7. Split payments across marketplace sellers and the platform (multi-party payments).

As a staff engineer, the core challenges at this scale are: **exactly-once payment execution** under high concurrency, **sharding for throughput** without losing transactional guarantees, **multi-region availability**, and a **ledger system** that can handle billions of rows while staying auditable.

# Requirements

## Functional Requirements

1. **Pay** — Accept a payment for an order. Supports credit/debit card, wallet (Apple Pay / Google Pay), and bank transfer via PSP.
2. **Refund** — Full or partial refund of a completed payment.
3. **Payment status query** — Get current state of a payment by order ID or payment ID.
4. **Webhook ingestion** — Receive and process async status updates from the PSP (e.g. settlement confirmation, dispute opened).
5. **Ledger** — Maintain a double-entry ledger that records every debit/credit.

**✍️ Note**: We are NOT designing the order service, the product catalog, or the shopping cart. We assume an upstream order service calls us with a confirmed order.

## Non-Functional Requirements

### Scale

- **~50M payment transactions/day** on peak days (Prime Day, Black Friday). Normal days ~5-8M/day.
- Average TPS: ~60-90. Peak TPS: **~3,000** (comparable to Adyen's 2,700 TPS Black Friday peak).
- Each payment produces 2-5 ledger entries → **100M-250M ledger rows/day** on peak days.
- Ledger grows to **~30B+ rows/year**. Must support archival and tiered storage.
- Hundreds of thousands of sellers → **high cardinality on merchant_id**, with heavy skew (top 1% of sellers generate 50%+ of volume).
- Multi-region: customers and sellers span US, EU, and APAC.

**✍️ Note**: At Amazon scale, the challenge is BOTH correctness AND throughput. We must shard from day one. Unlike ad serving (which tolerates approximate answers), every single payment must be exactly right — but we also can't afford a single-primary bottleneck at 3,000 TPS peak.

### Consistency

- **Exactly-once payment execution** — A retry of the same pay request must not create a duplicate charge. This is the single most important requirement.
- Strong consistency for payment state transitions (no phantom reads of stale state).
- Ledger must always balance (sum of debits == sum of credits).

### Availability

- **99.99% uptime** target → ~52 minutes of downtime per year max. Payments are revenue-critical.
- **Multi-region active-active** deployment. A regional outage should not take down payments globally.
- Graceful degradation: if one PSP is down, failover to a secondary PSP. If all PSPs are down, queue the intent and retry (do NOT lose the payment intent).
- **Multi-PSP strategy** is mandatory at this scale — a single PSP is a single point of failure.

### Latency

- Pay API: < 2s p99 (dominated by PSP round-trip, typically 300-800ms).
- Status query: < 50ms p99 (hot path, must be cached or single-shard lookup).
- Webhook processing: < 5s end-to-end.
- Merchant dashboard queries: < 500ms p99 (can tolerate slight staleness via read replicas or CQRS).

### Security / Compliance

- No raw card numbers touch our servers (PCI-DSS scope reduction via PSP tokenization).
- All API calls authenticated; internal service-to-service mTLS.
- Sensitive data (PII, tokens) encrypted at rest.

## Edge Cases

### Idempotency

The client (order service) sends a unique `idempotency_key` with every pay/refund request. If we receive the same key again, we return the cached result instead of creating a new charge.

### PSP timeout / ambiguous response

If the PSP call times out or returns a 5xx, we do NOT know if the charge went through. We must:
1. Mark the payment as `UNKNOWN`.
2. Poll the PSP or wait for a webhook to resolve the state.
3. Never retry the charge blindly — this could double-charge.

### Partial capture

Some business models authorize first, then capture later (e.g. hotel booking). The auth expires after 7 days (card network rule). We need to track auth expiry.

### Dispute / Chargeback

The PSP notifies us via webhook. We mark the payment as `DISPUTED`, debit the merchant's balance, and potentially trigger a review workflow.

### Webhook replay / out-of-order delivery

Webhooks can arrive out of order or be replayed. We use the payment state machine to reject invalid transitions (e.g. ignoring a `CAPTURED` webhook if we already processed `REFUNDED`).

---

# API Design

## External API (called by order service)

### Create Payment

```
POST /v1/payments
Headers: Idempotency-Key: <uuid>

Request Body:
{
  "order_id": "order_abc123",
  "amount": 5999,              // in smallest currency unit (cents)
  "currency": "USD",
  "payment_method_token": "pm_tok_xxx",  // PSP token from client-side
  "capture": true,             // auto-capture or auth-only
  "metadata": { "user_id": "u_123", "merchant_id": "m_456" }
}

Response 200:
{
  "payment_id": "pay_xyz789",
  "status": "CAPTURED",        // or "AUTHORIZED" if capture=false
  "psp_transaction_id": "pi_stripe_abc",
  "amount": 5999,
  "currency": "USD",
  "created_at": "2026-03-05T10:00:00Z"
}
```

**✍️ Note**: `payment_method_token` is created by the PSP's client-side SDK (Stripe Elements, Adyen Drop-in). Raw card numbers never reach our API.

### Refund Payment

```
POST /v1/payments/{payment_id}/refund
Headers: Idempotency-Key: <uuid>

Request Body:
{
  "amount": 2000,              // partial refund; omit for full refund
  "reason": "CUSTOMER_REQUEST"
}

Response 200:
{
  "refund_id": "ref_abc",
  "payment_id": "pay_xyz789",
  "status": "REFUND_PENDING",
  "amount": 2000
}
```

### Get Payment Status

```
GET /v1/payments/{payment_id}

Response 200:
{
  "payment_id": "pay_xyz789",
  "order_id": "order_abc123",
  "status": "CAPTURED",
  "amount": 5999,
  "currency": "USD",
  "refunded_amount": 2000,
  "psp_transaction_id": "pi_stripe_abc",
  "created_at": "...",
  "updated_at": "..."
}
```

## Internal API (PSP → Us)

### Webhook Endpoint

```
POST /v1/webhooks/psp
Headers: X-PSP-Signature: <hmac>

Body: (PSP-specific payload)
{
  "event_type": "payment.captured",
  "psp_transaction_id": "pi_stripe_abc",
  "amount": 5999,
  ...
}
```

We verify the HMAC signature, map the PSP event to our internal state machine, and update accordingly.

---

# High Level Design

```
                              ┌─────────────────────────────────────┐
                              │         Load Balancer / API GW      │
                              └──────────────┬──────────────────────┘
                                             │
┌─────────────┐     ┌────────────────────────▼───────────────────────────┐
│ Order Service│────▶│           Payment Service Fleet (stateless)        │
│  (upstream)  │     │  ┌──────────┐  ┌───────────┐  ┌────────────────┐  │
└─────────────┘     │  │  State   │  │   PSP     │  │   Idempotency  │  │
                    │  │  Machine │  │   Router  │  │   Checker      │  │
                    │  └──────────┘  └─────┬─────┘  └────────────────┘  │
                    └──────────┬───────────┼────────────────────────────┘
                               │           │
              ┌────────────────┼───────┐   │   ┌───────────────┐
              │                │       │   ├──▶│  PSP A (Stripe)│
              ▼                ▼       ▼   │   └───────────────┘
  ┌────────────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐
  │  Payment DB    │  │ Ledger DB│  │  Redis   │  │  PSP B (Adyen)│
  │  (Vitess,      │  │ (sharded │  │  Cluster │  └───────┬───────┘
  │   sharded by   │  │  by      │  │ (idempot)│          │
  │   payment_id)  │  │ acct_id) │  └──────────┘          │
  └───────┬────────┘  └────┬─────┘                        │
          │                │                    ┌─────────▼────────┐
          │    CDC / Outbox │                    │  Webhook Consumer│
          │    ─────────────┘                    │  Fleet           │
          │                                     └──────────────────┘
          ▼
  ┌────────────────┐     ┌───────────────────┐
  │  Elasticsearch │     │  Reconciliation   │
  │  (merchant     │     │  Worker (batch)   │
  │   dashboards)  │     └───────────────────┘
  └────────────────┘
```

### Payment Service Fleet

Stateless application tier (20-30+ instances behind a load balancer). Receives pay/refund requests, enforces idempotency, drives the state machine, calls the PSP via the PSP Router, and writes to the payment DB. Horizontally scalable — add instances to handle more TPS.

### PSP Router

Selects which PSP to use per transaction based on success rate, cost, region, and card network coverage. Provides automatic failover if a PSP is degraded. Implements the adapter/strategy pattern so each PSP's API specifics are abstracted away.

### Payment DB (Sharded PostgreSQL via Vitess)

Stores payment records with current state, order mapping, PSP references. **Sharded by `payment_id`** for uniform write distribution. 16+ shards, each shard is a PostgreSQL primary + replicas. Vitess VTGate handles routing.

### Ledger DB (Sharded PostgreSQL)

Append-only double-entry ledger. Every money movement is a pair of debit + credit entries. **Sharded by `account_id`** so that balance queries are single-shard. Separate shard cluster from Payment DB because different shard key.

### Redis Cluster (Idempotency Store)

Maps `idempotency_key → (status, response)` with a TTL (e.g. 24h). Used for fast de-duplication of retries. Redis Cluster auto-shards by key across hash slots.

### Webhook Consumer Fleet

Dedicated stateless workers that receive PSP webhooks, verify HMAC signatures, de-duplicate by `psp_event_id`, and apply state transitions. Separated from the main payment service to isolate webhook burst traffic (PSPs can replay large batches).

### Elasticsearch (CQRS Read Store)

Receives payment data via CDC (Change Data Capture) from the Payment DB outbox. Powers merchant dashboards and search queries ("show me all payments for merchant X last month") without scatter-gather across payment shards.

### Reconciliation Worker

Daily batch job that pulls settlement reports from each PSP and compares against our internal records. Catches discrepancies that the online system missed.

### 3rd Party PSPs

Stripe / Adyen / Braintree. Handle card tokenization, 3DS, charge execution, and settlement with card networks and issuing banks. We integrate with **at least 2 PSPs** for redundancy.

---

# Detailed Technical Design

## 1. Payment State Machine

This is the heart of the system. Every payment transitions through well-defined states. **Invalid transitions are rejected.**

```
                    ┌──────────────┐
                    │   CREATED    │
                    └──────┬───────┘
                           │ call PSP
                    ┌──────▼───────┐
              ┌─────│  PROCESSING  │─────┐
              │     └──────┬───────┘     │
           timeout         │          PSP error
              │         success          │
              ▼            ▼             ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ UNKNOWN  │ │AUTHORIZED│ │  FAILED  │
        └────┬─────┘ └────┬─────┘ └──────────┘
             │            │
        poll/webhook   capture
             │            │
             ▼            ▼
        (resolves)  ┌──────────┐
                    │ CAPTURED │
                    └────┬─────┘
                         │
              ┌──────────┼──────────┐
              ▼                     ▼
        ┌──────────┐          ┌──────────┐
        │ REFUNDED │          │ DISPUTED │
        │(full/    │          │          │
        │ partial) │          └──────────┘
        └──────────┘
```

**✍️ Note**: The state machine is enforced at the DB level. Every state update uses an optimistic lock:

```sql
UPDATE payments
SET status = 'CAPTURED', version = version + 1, updated_at = NOW()
WHERE payment_id = ? AND status = 'AUTHORIZED' AND version = ?;
-- If affected rows = 0, the transition is invalid → reject
```

This prevents race conditions between the synchronous API path and async webhook path trying to update the same payment concurrently.

## 2. Idempotency — Exactly-Once Execution

This is the most critical design element. Here's how it works:

### Step-by-step flow for `POST /v1/payments`:

```
1. Receive request with Idempotency-Key = "key_abc"
2. BEGIN TRANSACTION
3.   Look up "key_abc" in idempotency_keys table
4.   IF found:
       - IF status = COMPLETED → return cached response (HTTP 200)
       - IF status = IN_PROGRESS → return HTTP 409 (concurrent request)
5.   IF not found:
       - INSERT idempotency_keys (key, status=IN_PROGRESS, request_hash)
6.   INSERT payments (payment_id, status=CREATED, ...)
7. COMMIT TRANSACTION
8. Call PSP (this is OUTSIDE the transaction — network call)
9. BEGIN TRANSACTION
10.  Update payments status based on PSP response
11.  Write ledger entries (debit buyer, credit merchant)
12.  Update idempotency_keys status=COMPLETED, cached_response=<response>
13. COMMIT TRANSACTION
14. Return response
```

**❓ Why is the PSP call outside the transaction?**

Database transactions should not hold open during external network calls. The PSP call can take 300-800ms. Holding a DB transaction that long causes lock contention and connection pool exhaustion.

**❓ What if the service crashes between step 8 and step 9?**

The payment is stuck in `CREATED` state with the idempotency key `IN_PROGRESS`. A background reconciliation worker picks up stuck payments (CREATED for > 5 minutes), polls the PSP to check actual status, and resolves them.

**❓ What if the PSP call times out?**

We set the payment to `UNKNOWN` and do NOT retry the charge. Instead:
1. A scheduled poller queries the PSP: `GET /v1/payment_intents/{psp_id}`.
2. Or we wait for the PSP webhook to tell us the actual outcome.
3. Once resolved, we transition to `CAPTURED` or `FAILED`.

This prevents double-charging.

## 3. PSP Integration

### Tokenization flow (client-side)

```
┌──────────┐        ┌─────────┐        ┌──────────────┐
│  Browser /│        │   PSP   │        │   Payment    │
│  Mobile   │        │  (SDK)  │        │   Service    │
└─────┬─────┘        └────┬────┘        └──────┬───────┘
      │  1. Enter card     │                   │
      │  details in PSP    │                   │
      │  iframe/SDK        │                   │
      │───────────────────▶│                   │
      │                    │                   │
      │  2. Return token   │                   │
      │◀───────────────────│                   │
      │                    │                   │
      │  3. Submit order   │                   │
      │  + token           │                   │
      │────────────────────────────────────────▶│
      │                    │                   │
      │                    │  4. Charge using  │
      │                    │◀──────────────────│
      │                    │     token         │
      │                    │                   │
      │                    │  5. Charge result │
      │                    │──────────────────▶│
      │                    │                   │
      │  6. Payment result │                   │
      │◀───────────────────────────────────────│
```

**✍️ Note**: Card numbers go directly from the browser to the PSP. Our server only sees the token (e.g. `pm_tok_xxx`). This keeps us **out of PCI-DSS scope** for card data handling.

### PSP API call (server-side)

```python
# Pseudo code for PSP charge
def charge_via_psp(payment):
    try:
        result = psp_client.create_charge(
            amount=payment.amount,
            currency=payment.currency,
            payment_method=payment.token,
            idempotency_key=payment.payment_id,  # use our payment_id as PSP idempotency key
            metadata={"order_id": payment.order_id}
        )
        return PSPResult(success=True, txn_id=result.id, status=result.status)
    except PSPTimeout:
        return PSPResult(success=False, status="UNKNOWN")
    except PSPError as e:
        return PSPResult(success=False, status="FAILED", error=str(e))
```

**❓ Why do we pass our `payment_id` as the PSP's idempotency key?**

If our service crashes and retries the same charge, the PSP de-duplicates on their end and returns the original result. This is a **belt-and-suspenders** approach — idempotency on both our side AND the PSP side.

## 4. Webhook Processing

PSP webhooks are async notifications about events we may or may not have already processed.

### Design principles:

1. **Signature verification** — Every webhook is HMAC-signed by the PSP. We reject any webhook with an invalid signature.
2. **Idempotent processing** — The same webhook event can arrive multiple times. We use `psp_event_id` to de-duplicate.
3. **State machine enforcement** — We only apply valid transitions. If a `CAPTURED` webhook arrives for a payment already in `REFUNDED`, we log a warning and skip.
4. **At-least-once delivery** — PSPs guarantee at-least-once, not exactly-once. Our de-duplication handles this.

### Webhook processing flow:

```
1. Receive POST /v1/webhooks/psp
2. Verify HMAC signature → reject if invalid (HTTP 401)
3. Parse event, extract psp_transaction_id and event_type
4. Look up psp_event_id in webhook_events table
   - If found → return HTTP 200 (already processed)
5. Look up payment by psp_transaction_id
   - If not found → queue for retry (payment may not be created yet due to race)
6. Validate state transition (e.g. AUTHORIZED → CAPTURED is valid)
7. BEGIN TRANSACTION
   - Update payment state
   - Write ledger entries if needed
   - Insert webhook_events (psp_event_id) for de-dup
8. COMMIT
9. Return HTTP 200
```

**✍️ Note**: Always return HTTP 200 quickly. If processing takes too long, the PSP will retry, causing duplicate work. For heavy processing, ack immediately and process asynchronously via a queue.

## 5. Double-Entry Ledger

Every money movement is recorded as a balanced pair of entries. This is how we maintain an auditable, tamper-evident financial record.

### Ledger schema:

```sql
CREATE TABLE ledger_entries (
    entry_id        BIGSERIAL PRIMARY KEY,
    payment_id      UUID NOT NULL,
    account_id      VARCHAR(64) NOT NULL,   -- e.g. "buyer:u_123", "merchant:m_456", "platform:fees"
    entry_type      VARCHAR(16) NOT NULL,   -- DEBIT or CREDIT
    amount          BIGINT NOT NULL,        -- in cents, always positive
    currency        VARCHAR(3) NOT NULL,
    description     VARCHAR(256),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Index for account balance queries
CREATE INDEX idx_ledger_account ON ledger_entries (account_id, created_at);
```

### Example: Customer pays $59.99 for an order, platform takes 2.9% + $0.30 fee

| entry_id | payment_id | account_id | type | amount | description |
|---|---|---|---|---|---|
| 1 | pay_xyz | buyer:u_123 | DEBIT | 5999 | Order payment |
| 2 | pay_xyz | merchant:m_456 | CREDIT | 5795 | Order revenue |
| 3 | pay_xyz | platform:fees | CREDIT | 204 | Platform fee (2.9% + 30c) |

**✍️ Note**: sum(DEBIT) = sum(CREDIT) = 5999. The ledger always balances. This invariant can be checked by a daily batch job.

### Example: Partial refund of $20.00

| entry_id | payment_id | account_id | type | amount | description |
|---|---|---|---|---|---|
| 4 | pay_xyz | merchant:m_456 | DEBIT | 2000 | Partial refund |
| 5 | pay_xyz | buyer:u_123 | CREDIT | 2000 | Partial refund |

**❓ Why not just update a balance column?**

An append-only ledger is auditable. You can reconstruct the balance of any account at any point in time by replaying entries. A mutable balance column loses history and is harder to debug when something goes wrong.

**❓ How do we get the current balance of an account?**

```sql
SELECT
  SUM(CASE WHEN entry_type = 'CREDIT' THEN amount ELSE 0 END) -
  SUM(CASE WHEN entry_type = 'DEBIT' THEN amount ELSE 0 END) AS balance
FROM ledger_entries
WHERE account_id = 'merchant:m_456';
```

For hot accounts (millions of entries), we maintain a materialized `account_balances` table updated in the same transaction as ledger writes.

## 6. Database Schema (Payment DB)

```sql
CREATE TABLE payments (
    payment_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id            VARCHAR(64) NOT NULL,
    idempotency_key     VARCHAR(128) UNIQUE NOT NULL,
    status              VARCHAR(32) NOT NULL DEFAULT 'CREATED',
    amount              BIGINT NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    refunded_amount     BIGINT NOT NULL DEFAULT 0,
    payment_method_token VARCHAR(256),
    psp_transaction_id  VARCHAR(128),
    merchant_id         VARCHAR(64) NOT NULL,
    buyer_id            VARCHAR(64) NOT NULL,
    metadata            JSONB,
    version             INT NOT NULL DEFAULT 1,    -- optimistic lock
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_order ON payments (order_id);
CREATE INDEX idx_payments_psp_txn ON payments (psp_transaction_id);

CREATE TABLE idempotency_keys (
    idempotency_key     VARCHAR(128) PRIMARY KEY,
    status              VARCHAR(16) NOT NULL,       -- IN_PROGRESS, COMPLETED
    request_hash        VARCHAR(64) NOT NULL,       -- SHA256 of request body
    cached_response     JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at          TIMESTAMPTZ NOT NULL         -- TTL, e.g. created_at + 24h
);

CREATE TABLE webhook_events (
    psp_event_id        VARCHAR(128) PRIMARY KEY,
    payment_id          UUID NOT NULL,
    event_type          VARCHAR(64) NOT NULL,
    processed_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**❓ Why PostgreSQL (sharded)?**

- ACID transactions are essential for the payment state machine + ledger writes.
- JSONB for flexible metadata without schema migrations.
- At ~3,000 TPS peak, a single PostgreSQL primary is at its limit. We shard from the start using Vitess (or Citus).
- See **Scalability Deep Dive** below for detailed shard key selection and cross-shard consistency patterns.

**❓ How is the DB sharded?**

- **Payment DB** → sharded by `payment_id` (UUID, uniform distribution, no hot shards).
- **Ledger DB** → sharded by `account_id` (balance queries are single-shard).
- **Idempotency** → Redis Cluster by `idempotency_key`, with DB fallback.

The payment and ledger tables live on **different shard clusters** because they have different optimal shard keys. Cross-shard consistency is achieved via the **outbox pattern** (see Scalability section).

---

## 7. Reconciliation

Reconciliation is the process of comparing our internal records with the PSP's records (and ultimately the bank's settlement files) to catch discrepancies.

### Types of discrepancies:

1. **We charged, PSP has no record** — Possible if our PSP call succeeded but PSP lost the record (extremely rare).
2. **PSP charged, we have no record** — Our service crashed after PSP call but before DB write. The reconciliation worker fixes this.
3. **Amount mismatch** — Currency conversion issues or PSP applied a different fee.
4. **Status mismatch** — We think it's CAPTURED, PSP says REFUNDED (e.g. chargeback we missed).

### Reconciliation flow:

```
Daily Batch Job:
1. Pull settlement report from PSP (CSV/API)
2. For each PSP transaction:
   a. Match by psp_transaction_id to our payments table
   b. Compare amount, currency, status
   c. Flag discrepancies → write to reconciliation_exceptions table
3. For each unmatched internal payment (we have it, PSP doesn't):
   a. Flag for investigation
4. Generate reconciliation report
5. Alert on-call if exception count > threshold
```

**✍️ Note**: At staff level, you MUST mention reconciliation. It's the safety net that catches all the edge cases your online system misses. Real-world payment systems always have reconciliation jobs.

---

## 8. Failure Handling & Retry Strategy

| Scenario | Action |
|---|---|
| PSP returns 4xx (bad request, card declined) | Mark payment FAILED. Do NOT retry. |
| PSP returns 5xx (server error) | Mark payment UNKNOWN. Poll PSP for status. |
| PSP call timeout | Mark payment UNKNOWN. Do NOT retry charge. Poll or wait for webhook. |
| Our DB write fails after PSP success | Retry DB write with backoff. Payment is safe (PSP idempotency key protects against double-charge). |
| Webhook signature invalid | Reject (HTTP 401). Log for security audit. |
| Webhook for unknown payment_id | Queue for retry with backoff (payment record may not be committed yet). |

**❓ Why do we NEVER blindly retry a charge on timeout?**

Because the PSP may have actually processed it. Retrying would create a second charge. Instead, we query the PSP to check the actual status, or rely on the PSP's webhook to tell us. This is the **cardinal rule** of payment engineering.

---

# Other Considerations

## Multi-PSP failover

At scale, relying on a single PSP is a single point of failure. We can integrate 2-3 PSPs and route traffic based on:
- **Success rate** — If PSP A's success rate drops below a threshold, route to PSP B.
- **Cost** — Different PSPs have different fee structures per region/card type.
- **Card network coverage** — Some PSPs have better coverage in certain regions.

The payment service would have a PSP router that selects the optimal PSP per transaction. The PSP-specific logic is behind an adapter/strategy pattern.

## Currency handling

- All amounts stored in **smallest currency unit** (cents for USD, yen for JPY).
- Currency conversion happens at the PSP layer; we store both original and settled amounts.
- Be careful: JPY and KRW have no fractional units (1 JPY = 1 unit, not 100).

## Monitoring & Alerting

Key metrics:
- **Payment success rate** (by PSP, by payment method, by region) — alert if drops below 95%.
- **PSP latency** p50/p95/p99.
- **Webhook processing lag** — alert if > 1 minute behind.
- **Reconciliation exception count** — alert if > 0.
- **Ledger balance check** — daily sum(debit) vs sum(credit) assertion.

## Scalability Deep Dive

At Amazon-scale (~50M payment transactions/day peak, ~3,000 TPS burst), sharding is not optional — it's a core architectural requirement from day one.

| Component | Scale | Architecture |
|---|---|---|
| Payment Service | ~3,000 TPS peak | 20-30 stateless instances |
| Payment DB | ~50M rows/day peak | Vitess, 16 shards by `payment_id` |
| Ledger DB | ~250M rows/day peak | Separate cluster, 16 shards by `account_id` |
| Idempotency | ~3,000 checks/sec peak | Redis Cluster (6+ nodes) |
| Webhook ingestion | Burst-heavy | Dedicated consumer fleet + SQS/Kafka |
| Merchant dashboards | Scatter-gather too slow | Elasticsearch via CDC |

### Sharding the Payment DB

**Shard key: `payment_id`**

Why `payment_id` and not `merchant_id` or `buyer_id`?

| Shard key candidate | Pros | Cons |
|---|---|---|
| `payment_id` (UUID) | Even distribution, no hot shards, primary lookup is single-shard | Looking up "all payments for merchant X" requires scatter-gather |
| `merchant_id` | All payments for one merchant on one shard, good for merchant dashboards | Hot merchant problem — a merchant doing 50% of volume creates a hot shard |
| `buyer_id` | All payments for one buyer on one shard | Same hot-key problem for power buyers; most queries are by payment_id not buyer_id |

**We choose `payment_id`** because:
1. Payment creation and state updates are the hottest path — and they're always by `payment_id`.
2. UUID-based keys give near-perfect uniform distribution across shards.
3. The hot-merchant problem is real. Amazon on your platform could skew one shard to 100x the load of others.

**❓ How do we handle "get all payments for merchant X" after sharding by payment_id?**

This is a **scatter-gather** query — the API server fans out to all shards, each shard filters by `merchant_id`, and the API server merges results. This is acceptable because merchant dashboard queries are:
- Low frequency (dashboard, not checkout path)
- Paginated (LIMIT 50 per page)
- Cacheable (merchant dashboards can tolerate 5-10s staleness)

For high-volume merchants that need fast dashboard access, we can maintain a **secondary index table** (or a read-optimized view in a separate store like Elasticsearch):

```
merchant_payments_index:
  merchant_id → [payment_id_1, payment_id_2, ...]
```

This index is updated asynchronously via CDC (Change Data Capture) from the payment DB.

**❓ How do we route requests to the correct shard?**

Consistent hashing on `payment_id`:

```
shard_id = hash(payment_id) % num_shards
```

We use a shard routing layer (e.g., Vitess VTGate, Citus coordinator, or a custom proxy). The application code is ideally shard-unaware — it talks to the proxy as if it's a single database.

For Vitess-style sharding:

```sql
-- VSchema definition
CREATE VINDEX payment_hash USING hash;

CREATE TABLE payments (
    payment_id UUID,
    ...
) VINDEXED ON (payment_id) USING payment_hash;
```

**❓ How many shards?**

Start with a power of 2 (e.g., 16 shards). At 100M payments/day:
- ~1,200 TPS avg, ~6,000 TPS peak
- 16 shards → ~375 TPS peak per shard (comfortable for PostgreSQL)
- Each shard holds ~6M payments/day → ~2.2B payments/year per shard

Over-sharding upfront (e.g., 256 logical shards mapped to 16 physical nodes) gives you room to split without resharding later.

### Sharding the Ledger DB

The ledger is the first table to hit scale limits because each payment produces 2-5 ledger entries. At 100M payments/day, that's **200M-500M rows/day**.

**Shard key: `account_id`**

Why `account_id` and not `payment_id`?

- The most critical ledger query is **"what is the balance of account X?"** — this sums all entries for one `account_id`.
- If we shard by `payment_id`, a balance query requires scatter-gather across ALL shards (every shard has entries for the same merchant). This is unacceptable for a hot-path query.
- Sharding by `account_id` puts all entries for one account on one shard → balance is a single-shard query.

```
shard_id = hash(account_id) % num_shards
```

**❓ Doesn't this create hot shards for high-volume merchants?**

Yes, a merchant processing 10M transactions/day will have a hot shard. Mitigations:

1. **Account splitting** — Split a hot merchant's ledger into sub-accounts (e.g., `merchant:m_456:shard_0` through `merchant:m_456:shard_7`). The payment service round-robins writes across sub-accounts. Balance query sums across all sub-accounts (fan-out of 8 is acceptable).

2. **Materialized balance table** — Instead of summing all ledger entries on every query, maintain a `account_balances` table updated in the same transaction as each ledger write:

```sql
-- In the same transaction as ledger INSERT:
UPDATE account_balances
SET balance = balance + <delta>, version = version + 1
WHERE account_id = ?;
```

This reduces the balance query to a single-row lookup, regardless of how many ledger entries exist.

3. **Time-based compaction** — Periodically roll up old ledger entries into summary rows. For example, collapse all entries for merchant X from January 2025 into a single "opening balance" entry. The raw entries move to cold storage (S3 / archival DB) for audit purposes.

### Sharding the Idempotency Store

**Option A: Redis Cluster** — Shard by `idempotency_key` using Redis Cluster's hash slots. Simple, fast, and TTL-based expiry is built-in. This is the preferred approach.

**Option B: Colocate with Payment DB** — If idempotency keys are stored in the `idempotency_keys` table, they naturally shard with the payment DB. But this requires the idempotency key to map to the same shard as the payment, which is tricky since we don't know the `payment_id` yet when we first check the idempotency key.

**Solution**: Use Redis for the idempotency fast-path (check + reserve), and write to the DB table as a durable backup in the same transaction as the payment creation:

```
1. Check Redis: does idempotency_key exist?
   - Yes → return cached response
   - No → SET idempotency_key IN_PROGRESS (with NX + TTL)
2. Create payment in DB (idempotency_key stored in payments table)
3. After PSP call, update Redis with cached response
```

If Redis loses the key (eviction, failover), the DB table acts as fallback. This is acceptable because Redis misses just cause a redundant DB lookup, not a double-charge.

### Cross-Shard Transaction Problem

After sharding, we lose the ability to do atomic cross-shard transactions. This matters for the payment flow because steps 6-12 of the idempotency flow (update payment + write ledger + update idempotency key) may now span shards.

**Solutions:**

1. **Colocate payment + ledger on the same shard** — If we shard both by `payment_id`, they land on the same shard and we keep ACID. But this breaks ledger balance queries (see above).

2. **Saga pattern with outbox** — The preferred approach at scale:

```
Payment Shard:
  BEGIN TX
    UPDATE payments SET status = CAPTURED
    INSERT INTO outbox (event_type=PAYMENT_CAPTURED, payment_id, amount, ...)
  COMMIT

Async event processor (reads outbox via CDC):
  → Write ledger entries to Ledger Shard
  → Update idempotency cache
```

The outbox pattern guarantees that the event is published if and only if the payment update commits. The ledger write is eventually consistent but guaranteed to happen. If the ledger write fails, the event processor retries.

**✍️ Note**: This is a **classic staff-level trade-off**. We trade strong consistency (single-DB ACID) for scalability (sharded DBs + eventual consistency via outbox/saga). The reconciliation job (section 7) catches any drift.

3. **Two-phase commit (2PC)** — Technically possible but adds latency and a coordinator as a single point of failure. Not recommended for payment systems at scale.

### Read Scaling

Reads are simpler to scale than writes:

- **Read replicas** — Each shard has 2+ read replicas. Route read-only queries (status lookups) to replicas. Acceptable for queries that tolerate ~100ms replication lag.
- **Caching** — Cache payment status in Redis with short TTL (30s). Cache hit rate is high because most status queries happen right after payment creation (order confirmation page). At 3,000 TPS, caching avoids ~90% of DB reads.
- **CQRS with Elasticsearch** — Merchant dashboard queries ("show me all payments in the last 30 days, filtered by status and amount") would require scatter-gather across all 16 payment shards. Instead, we replicate payment data to Elasticsearch via CDC. The dashboard service queries ES directly — fast, filterable, and doesn't load the OLTP shards.

### Multi-Region Architecture

At Amazon scale, a single-region deployment is unacceptable — both for latency (users are global) and availability (a regional AWS outage takes down payments).

```
┌──────────────────────────────────────────────────────────────┐
│                    Global DNS / Load Balancer                │
│              (Route53 latency-based routing)                 │
└────────┬──────────────────┬──────────────────┬───────────────┘
         │                  │                  │
    ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
    │ US-EAST │        │ EU-WEST │        │ AP-EAST │
    │ Region  │        │ Region  │        │ Region  │
    │         │        │         │        │         │
    │ Payment │        │ Payment │        │ Payment │
    │ Service │        │ Service │        │ Service │
    │ Fleet   │        │ Fleet   │        │ Fleet   │
    │         │        │         │        │         │
    │ Payment │        │ Payment │        │ Payment │
    │ DB Shard│        │ DB Shard│        │ DB Shard│
    │ Cluster │        │ Cluster │        │ Cluster │
    │         │        │         │        │         │
    │ Ledger  │        │ Ledger  │        │ Ledger  │
    │ DB Shard│        │ DB Shard│        │ DB Shard│
    │ Cluster │        │ Cluster │        │ Cluster │
    │         │        │         │        │         │
    │ Redis   │        │ Redis   │        │ Redis   │
    │ Cluster │        │ Cluster │        │ Cluster │
    └────┬────┘        └────┬────┘        └────┬────┘
         │                  │                  │
         └──────────────────┼──────────────────┘
                            │
              ┌─────────────▼─────────────┐
              │  Cross-Region Reconciler  │
              │  (async, eventual)         │
              └───────────────────────────┘
```

**Design: Region-local, NOT cross-region replication**

Each region is an **independent, self-contained stack**. A payment created in US-EAST stays in US-EAST for its entire lifecycle. We do NOT replicate payment data across regions because:

1. **Cross-region replication of transactional data adds dangerous latency** to writes (~100-200ms round-trip US↔EU), which is unacceptable for payment state machine transitions that require strong consistency.
2. **Conflict resolution** for payment state machines is extremely risky — two regions concurrently updating the same payment could lead to double-charges or phantom refunds.
3. **Data residency laws** (EU GDPR, India RBI) may require payment data to stay in-region.

**❓ How do we route a payment to the correct region?**

The order service tags each payment with a `region` based on the buyer's location (or the merchant's home region for marketplace sellers). DNS/load balancer routes the request to the correct regional stack.

```
payment_id format: pay_{region}_{uuid}
Example:  pay_us_east_a1b2c3d4
```

This way, any service that sees a `payment_id` knows which region to query.

**❓ What about cross-region queries?**

A merchant based in the US with buyers in EU needs a unified dashboard. This is handled by:
1. Each region publishes payment events to a **global event stream** (cross-region Kafka / Kinesis).
2. A global Elasticsearch cluster ingests events from all regions.
3. The merchant dashboard service queries the global ES cluster.

This is eventually consistent (seconds of delay), which is fine for dashboards.

**❓ What about regional failover?**

If US-EAST goes down:
1. **New payments** from US buyers are routed to US-WEST (backup region). The DNS failover takes ~30-60s.
2. **In-flight payments** in US-EAST are stuck until the region recovers. The reconciliation worker resolves them against the PSP.
3. We do NOT try to "move" an in-flight payment to another region — the state machine is too delicate for that. The PSP's idempotency key protects against double-charges when the region comes back.

### Ledger Tiered Storage

At ~250M ledger rows/day peak, the ledger grows to **~30B+ rows/year**. We can't keep all of this in hot PostgreSQL storage forever.

```
┌──────────────────────────────────────────────────────────┐
│                     Ledger Storage Tiers                  │
├──────────┬───────────────────┬────────────────────────────┤
│  HOT     │  0-90 days        │  Sharded PostgreSQL        │
│          │                   │  Full query capability      │
│          │                   │  ~7.5B rows per quarter     │
├──────────┼───────────────────┼────────────────────────────┤
│  WARM    │  90 days - 2 yrs  │  ClickHouse / TimescaleDB  │
│          │                   │  Compressed, columnar       │
│          │                   │  Aggregate queries only     │
├──────────┼───────────────────┼────────────────────────────┤
│  COLD    │  2+ years         │  S3 + Parquet files         │
│          │                   │  Audit/compliance access    │
│          │                   │  ~90% storage cost reduction│
└──────────┴───────────────────┴────────────────────────────┘
```

- A **nightly compaction job** rolls up individual ledger entries into daily/monthly summary rows before moving them to warm storage.
- Cold storage retains raw entries for regulatory compliance (7-year retention for financial records in most jurisdictions).
- The balance query uses the materialized `account_balances` table (hot tier only) — no need to scan warm/cold storage for current balances.

---

# Summary

The core of this design at Amazon scale:

1. **Idempotency** at both our layer (idempotency_key table + Redis Cluster) and the PSP layer (passing our payment_id as PSP's idempotency key). This guarantees exactly-once payment even across retries, crashes, and regional failovers.
2. **State machine** with optimistic locking to enforce valid transitions and handle concurrent updates from sync API + async webhooks.
3. **Sharding from day one** — Payment DB by `payment_id` (uniform distribution), Ledger DB by `account_id` (balance queries are single-shard), using Vitess for transparent routing.
4. **Cross-shard consistency via outbox pattern** — the classic staff-level trade-off: eventual consistency between payment and ledger shards, with reconciliation as the safety net.
5. **Multi-region active-active** — each region is an independent stack; no cross-region replication of transactional data. Global dashboards via CDC to Elasticsearch.
6. **Multi-PSP failover** — at least 2 PSPs with intelligent routing for availability, cost, and success rate optimization.
7. **Double-entry ledger** with tiered storage (hot/warm/cold) for auditability at 30B+ rows/year.
8. **Reconciliation** as the ultimate safety net — catches everything the online system misses.
9. **Never blindly retry a charge on timeout** — the cardinal rule of payments.
