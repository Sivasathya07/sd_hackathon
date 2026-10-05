# Payment Success, Then Order Failure

## 1. Core rule
Once money is taken, the customer must end with either an **order** or a **refund**, never with nothing.

## 2. Design choices
| Choice | Reason |
|---|---|
| Payment and event committed in one transaction (outbox) | No state where money is taken but the event is lost |
| Payment API returns 202 after commit | An Order Service outage never makes a successful payment look failed |
| Order creation is asynchronous | Isolates payment from Order Service availability |
| Idempotent order creation | Redelivery cannot create two orders |
| Reconciler as safety net | Repairs even if broker, consumer and DLQ all fail |
| Idempotent refund as last resort | A retried refund never refunds twice |

## 3. Flow
1. Customer calls POST /payments with reservation_id and Idempotency-Key.
2. Payment Service inserts a PENDING payment with a unique transaction_ref.
3. Gateway charge succeeds.
4. In **one local transaction**: payment becomes SUCCESS and a PaymentConfirmed outbox event is inserted.
5. Customer receives 202 (payment confirmed, order being created).
6. Outbox publisher sends PaymentConfirmed to the broker.
7. Order Service is down. Consumer retries with exponential backoff and jitter (max 8 attempts). The circuit breaker opens. The event stays safe in the queue.
8. When the Order Service recovers: breaker goes half-open, 3 probes succeed, backlog drains.
9. Consumer checks processed_events, creates the order and records event_id in one transaction, publishes OrderCreated.
10. Notification: order confirmed.

## 4. Timeline for the 30 second outage (assumed timings)
| Time | What happens | Customer sees |
|---|---|---|
| T+0 s | Gateway charge succeeds, payment and outbox event committed | "Payment received" |
| T+0 to 30 s | Order Service down, retries at 1, 2, 4, 8, 16 s with jitter, breaker opens | "Order is being created" |
| T+30 s | Order Service recovers, breaker half-open, 3 probes succeed, closes | Same |
| T+35 s | Event processed once, order created | "Order confirmed" |

Inventory is unaffected. The unit stays CONFIRMED the whole time.

## 5. If retries are exhausted
1. Message moves to the DLQ and an alert fires.
2. The reconciler scans payments that are SUCCESS with no order after 5 minutes and re-publishes PaymentConfirmed.
3. If the order exists after re-publish, done.
4. If still failing after a hard cap of 30 minutes, the reconciler issues a refund using refund_ref derived from transaction_ref, marks the payment REFUNDED and the reservation RELEASED, and notifies the customer.

## 6. Other payment outcomes (for completeness)
| Scenario | Response |
|---|---|
| Payment fails | Release reservation (reserved -1, available +1), notify customer |
| Payment times out | Outcome unknown. Keep PENDING, query the gateway by transaction_ref, never re-charge blindly |
| Duplicate payment request | Same Idempotency-Key and transaction_ref return the same result |
| Payment Service crashes after charge, before commit | Row is PENDING with known transaction_ref. Reconciler finds the charge and moves it to SUCCESS |

## 6.1 Reservation and late payment
A reservation in PAYMENT_PENDING is not swept until payment timeout plus a grace period. If payment succeeds after release, the system tries to re-reserve. If stock is gone, it auto-refunds. We never sell a unit that does not exist.

## 7. Order state interaction
Order states: CREATED -> PAYMENT_PENDING -> CONFIRMED -> PROCESSING -> SHIPPED -> OUT_FOR_DELIVERY -> DELIVERED. Consumers check current state before applying a transition, so stale or repeated events cannot move an order backwards.

## 8. Trade-offs
- The order appears seconds later (eventual consistency).
- Needs an outbox publisher, reconciler and DLQ replay tool.
- This is the price for resilience and no lost payments.
