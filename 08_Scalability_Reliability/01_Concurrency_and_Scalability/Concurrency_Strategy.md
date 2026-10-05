# Concurrency Strategy


## 1. Problem
Product X has 100 units. 10,000 customers press "Buy Now" at the same time.
If many requests read "stock = 100" and each subtracts 1, we oversell.
We need **check stock + decrement stock** to be one indivisible step.

## 2. Core principle
Contention is controlled in one place: the atomic inventory operation.
Everything before it exists so that only about 100 to 300 requests ever reach it.
Load on the inventory path scales with **stock (100)**, not with **traffic (10,000 or 500,000)**.

## 3. Approaches compared

| Approach | How it works | Problem at 10,000 vs 100 | Verdict |
|---|---|---|---|
| Pessimistic lock (`SELECT ... FOR UPDATE`) | Lock the row, read, update, unlock | 10,000 transactions queue on one row, connection pool exhausted, long tail latency | Rejected |
| Optimistic lock (version column) | `UPDATE ... WHERE version = ?`, retry on failure | About 9,999 conflicts per round, retry storm | Rejected |
| Atomic conditional update (DB only) | `UPDATE ... SET available = available - 1 WHERE available >= 1` | Correct, but one hot row handles about 1,000 to 3,000 updates per second | Authoritative layer |
| Redis Lua atomic gate | Single-threaded script checks idempotency key and stock, then decrements | About 100,000 operations per second, no retries, but not durable | Fast layer |
| Single-writer queue per SKU | One consumer per product processes requests in order | Deterministic, but adds latency and operational complexity | Not selected |

## 4. Decision
**Redis Lua atomic gate (fast path) + PostgreSQL atomic conditional update (authoritative).**
The `version` column is kept for audit and to detect unexpected writers. It is not the main control.

- Redis is the bouncer: it removes about 9,900 requests cheaply.
- PostgreSQL is the cashier: it keeps the real books and has the final say.

**Safety rule:** Redis can only be more conservative than the database, never more generous.
If Redis wrongly says "sold out", we lose a few sales briefly and the reconciler fixes it.
If Redis wrongly says "available", the database rejects the request (0 rows updated, plus the CHECK constraint).

## 5. Where "no overselling" is guaranteed
In the PostgreSQL inventory row, inside one ACID transaction. Not in application code.

```sql
CREATE TABLE inventory (
  inventory_id       BIGINT PRIMARY KEY,
  product_id         BIGINT NOT NULL UNIQUE REFERENCES product(product_id),
  total_quantity     INT NOT NULL,
  available_quantity INT NOT NULL,
  reserved_quantity  INT NOT NULL,
  sold_quantity      INT NOT NULL,
  version            BIGINT NOT NULL DEFAULT 0,
  updated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (available_quantity >= 0 AND reserved_quantity >= 0 AND sold_quantity >= 0),
  CHECK (available_quantity + reserved_quantity + sold_quantity = total_quantity)
);
```

Reserve transaction (one local ACID transaction):

```sql
UPDATE inventory
   SET available_quantity = available_quantity - 1,
       reserved_quantity  = reserved_quantity + 1,
       version = version + 1
 WHERE product_id = :p AND available_quantity >= 1;   -- 0 rows => SOLD_OUT
INSERT INTO inventory_reservation (...);
INSERT INTO inventory_ledger (...);
INSERT INTO outbox (event: ReservationCreated);
```

Invariant: `available + reserved + sold = total`, always.

## 6. Last-unit scenario
Two requests reach the last unit at the same instant. The row lock serialises the two UPDATE statements.
The first updates 1 row. The second sees `available = 0`, updates 0 rows, and receives `409 SOLD_OUT`.
Even if the application had a bug, the CHECK constraint would abort the transaction.

## 7. Duplicate requests (2 percent, about 200)
1. The client sends an `Idempotency-Key` (a UUID created when Buy Now is clicked).
2. The Lua script checks the key first. A hit returns the same reservation and consumes no stock.
3. Final backstops in the database: `UNIQUE(idempotency_key)` and the unique active-reservation index on `(customer_id, sale_id)`.

## 8. Reservation lifecycle
`AVAILABLE -> RESERVED -> PAYMENT_PENDING -> CONFIRMED -> SOLD`
Failure paths: `RESERVED -> TIMEOUT -> RELEASED`, `PAYMENT_PENDING -> PAYMENT_FAILED -> RELEASED`.
Every transition is a conditional update (`WHERE status = 'RESERVED'`), so a double release cannot double-increment stock.
Reservation TTL is 5 minutes (assumption). A sweeper uses `SELECT ... FOR UPDATE SKIP LOCKED` so workers do not collide.

## 9. Failure behaviour
| Failure | Response |
|---|---|
| Redis down | Fall back to DB conditional update with tighter admission. Rebuild the counter from the DB afterwards. |
| DB primary down | Fail closed: 503 with Retry-After. Never sell on Redis alone. |
| Crash between Redis decrement and DB commit | Compensating INCR plus the reconciler (see 06_Reconciliation). |
| Service crash mid-request | Retry with the same Idempotency-Key is safe. |

## 10. Trade-offs
- Two stores (Redis and PostgreSQL) add reconciliation complexity, in exchange for burst absorption.
- Consistency is chosen over availability for inventory: some users get 503 during failover rather than risking oversell.
- A 5 minute TTL can hold stock briefly. A waitlist mitigates this.

