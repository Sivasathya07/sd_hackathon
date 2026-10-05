# Failure Matrix

Each row follows: failure -> detection -> impact -> response -> recovery -> guarantee kept.

| Component | Failure | Detection | Impact | Response | Recovery | Guarantee kept |
|---|---|---|---|---|---|---|
| Redis | Down, failover, data loss | Command timeout, health check | Fast gate lost, DB load rises | Fall back to DB conditional update, tighter admission cap | Rebuild counter from `inventory`, reconciler compares | No oversell (DB enforces) |
| PostgreSQL primary | Crash, network partition | Health check, replication lag alert | Reservations and writes impossible | Fail closed: 503 with Retry-After | Fence old primary, promote replica, resume sweeper and outbox | Consistency over availability |
| Payment gateway | Timeout, outage, 5xx | Timeout, error-rate window | Payments cannot complete | Circuit breaker opens, reservation held within cap | Half-open probes, reconcile pending by transaction_ref | No duplicate charge |
| Order Service | Unavailable (30 s test case) | Consumer failures, queue backlog, paid-without-order alert | Orders delayed, not lost | Payment stays CONFIRMED, event retried | Idempotent order creation, refund if cannot complete | Paid customer gets an order or a refund |
| Message broker | Down or lagging | Publish failures, consumer lag | Async steps delayed | Outbox keeps events in the DB | Drain outbox, DLQ for poison messages | No lost events |
| Gateway / Reservation instance | Process crash | Load balancer health check | Some requests fail | Remove from pool, autoscale | Client retries with same Idempotency-Key | Idempotent retry is safe |
| Reconciler | Stops running | Missing heartbeat for 2 minutes | Drift and stuck states not repaired | Alert, restart, leader election picks another | Next cycle repairs everything | Correctness does not depend on it (DB still enforces) |
| Sweeper | Stops running | Count of RESERVED past expires_at grows | Stock held longer than TTL | Alert, restart, other workers use SKIP LOCKED | Release overdue reservations idempotently | No double release |
| Network or dependency slow | Latency spike | p99 latency alert | Thread pile-up | Timeouts, bulkheads, load shedding | Auto-recovers when latency normalises | Degrades, does not collapse |
| Duplicate request | Client retry or double click | Idempotency key hit | None if handled | Return the same result | n/a | One reservation, one charge, one order |
| Notification provider | Email or SMS down | Send failures | Customer not informed | Retry, DLQ | Replay from DLQ | Orders unaffected |

## Severity guide
| Severity | Meaning | Example |
|---|---|---|
| Critical (page) | Money or inventory correctness at risk | Invariant breach, paid-without-order, DB failover |
| Warning (chat) | Degradation, no data risk yet | Queue lag, breaker open, DLQ not empty |



>
