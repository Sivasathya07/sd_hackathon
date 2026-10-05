# Failure Handling

## 1. Philosophy
1. **Every failure has a detector.** Without detection there is no recovery.
2. **Inventory fails closed, browsing fails open.** A wrong stock number is worse than a short pause.
3. **No event is lost.** The outbox pattern writes the event in the same transaction as the business change.
4. **Money is never lost.** A paid customer ends with an order or a refund.
5. **Redis is replaceable.** Losing it costs speed, never correctness.
6. **Degrade, do not collapse.** Timeouts, bulkheads, circuit breakers and load shedding stop one failure spreading.

## 2. Failure strategies by component
| Component | Strategy | Summary |
|---|---|---|
| Redis | Fall back | Use DB conditional update with tighter admission, rebuild counter afterwards |
| PostgreSQL primary | Fail closed | 503 with Retry-After, fence old primary, promote synchronous replica |
| Payment gateway | Degrade and hold | Circuit breaker, hold reservation within a cap, reconcile by transaction_ref |
| Order Service | Delay, do not fail | Payment stays CONFIRMED, event retried, idempotent order creation |
| Message broker | Outbox buffer | Events stay in the outbox table, publisher drains on recovery |
| Gateway or Reservation instance | Replace | Load balancer removes it, autoscaler adds one, client retries with same key |
| Slow dependency | Isolate | Timeouts, bulkheads, load shedding |

## 3. Cross-cutting mechanisms
| Mechanism | Purpose | Details in |
|---|---|---|
| Timeouts | Never wait forever | 04_Retry_and_DLQ/Retry_Strategy.md |
| Retries with backoff and jitter | Recover from transient errors | 04_Retry_and_DLQ/Retry_Strategy.md |
| Circuit breaker | Fail fast on sustained failure | 03_Circuit_Breaker/ |
| Dead-letter queue | Isolate poison messages | 04_Retry_and_DLQ/Dead_Letter_Queue.md |
| Idempotency | Make repeats harmless | Concurrency_Strategy.md and Dead_Letter_Queue.md |
| Reconciliation | Repair drift and stuck states | 06_Reconciliation/ |
| Compensation (refund, release) | Undo partial workflows | 05_Payment_Order_Recovery/ |

## 4. Database failure (fail closed)
1. Primary crashes. In-flight transactions roll back.
2. The reservation service times out (2 s assumption). The DB breaker opens and requests get 503 with Retry-After.
3. The failover manager confirms failure (health check plus quorum), **fences** the old primary, and promotes the synchronous replica.
4. Connection endpoint switches. Sweeper and outbox publisher resume. The breaker closes.
5. The reconciler verifies the invariant and rebuilds the Redis counter.

Targets (assumptions): RTO under 60 seconds, RPO zero for committed data (synchronous replication).
What keeps working: product browsing (CDN and cache). What stops: new reservations.

## 5. Payment gateway failure (degrade and hold)
1. The payment row is written as PENDING with a unique transaction_ref **before** calling the gateway.
2. Timeout means "unknown", not "failed". We never re-charge blindly.
3. The breaker opens. The reservation TTL is extended up to a cap (15 minutes assumption).
4. The reconciler asks the gateway for the status by transaction_ref and updates our records.
5. We do not fail over to a second gateway mid-payment because it risks a double charge. A second provider is only used for new attempts.

## 6. Order Service failure
See 05_Payment_Order_Recovery/Payment_Success_Order_Failure.md.

## 7. Trade-offs
- Fail-closed inventory means short unavailability during failover.
- Holding reservations during a gateway outage keeps stock locked longer.
- The outbox, reconciler and DLQ add components to operate, in exchange for no lost events or money.

