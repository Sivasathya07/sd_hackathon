# 08 - Scalability and Reliability Design



## Purpose
How SALESTORM stays correct and available when 10,000 customers compete for 100 units, and what happens when a component fails.

Core guarantees:
- No oversell: 100 units never produce more than 100 sales.
- No duplicate effects: repeated requests or messages never create duplicate reservations, charges or orders.
- No lost money: every successful payment ends in an order or a refund.

## Structure
| Folder | Files | Topic |
|---|---|---|
| 01_Concurrency_and_Scalability | Concurrency_Strategy.md, High_Traffic_Scaling.md, Throttling_and_Queueing.md, High_Concurrency_Architecture.png | Inventory contention, scaling to 50x, throttling and queueing |
| 02_Failure_Handling | Failure_Handling.md, Failure_Matrix.md, Failure_Handling_Flow.png | Failure strategy per component, detection and recovery |
| 03_Circuit_Breaker | Circuit_Breaker_Strategy.md, Circuit_Breaker_State_Diagram.png | Fail fast on gateway and Order Service failure |
| 04_Retry_and_DLQ | Retry_Strategy.md, Dead_Letter_Queue.md, Retry_and_DLQ_Flow.png | Timeouts, backoff with jitter, DLQ, idempotent consumers |
| 05_Payment_Order_Recovery | Payment_Success_Order_Failure.md, Payment_Success_Order_Failure_Recovery.png | Payment success followed by Order Service failure |
| 06_Reconciliation | Reconciliation_Strategy.md, Reconciliation_Flow.png | Repairing Redis, DB, payment and order drift |
| 07_Stress_Testing | Stress_Test_Scenarios.md, Expected_Results.md, Test_Results.png | Scenarios, expected outcomes, recorded results |

## Key design decisions
1. Concurrency: Redis Lua gate (fast path) plus PostgreSQL atomic conditional update (authoritative). Redis can only be stricter than the database, never looser.
2. Database failure: fail closed (503). Consistency over availability.
3. Async workflows: outbox pattern, at-least-once delivery, idempotent consumers, giving an exactly-once effect.
4. External dependencies: timeouts, retries with exponential backoff and jitter, circuit breaker.
5. Safety net: a reconciler repairs drift. PostgreSQL is the source of truth.

Full reasoning is recorded in 10_ADR/.

## Reading order
1. Concurrency_Strategy.md
2. Throttling_and_Queueing.md and High_Traffic_Scaling.md
3. Failure_Handling.md and Failure_Matrix.md
4. Circuit_Breaker_Strategy.md, Retry_Strategy.md, Dead_Letter_Queue.md
5. Payment_Success_Order_Failure.md
6. Reconciliation_Strategy.md
7. Stress_Test_Scenarios.md and Expected_Results.md

## Assumptions
All numeric values (timeouts, thresholds, retry counts, RTO and RPO, TTLs, intervals) are design assumptions to be tuned with load testing. They are not measured results. Expected_Results.md separates expected values from actual measured values.

## Related folders
- 02_HLD/ (failover design feeds the deployment diagram)
- 03_LLD/ (circuit breaker and consumer dedupe classes)
- 04_Database/ (processed_events, outbox, ledger tables)
- 05_API/ (503 Retry-After, 429, payment pending responses)
- 09_Security_Observability/ (alerts that detect each failure)
- 10_ADR/ (decision records)

