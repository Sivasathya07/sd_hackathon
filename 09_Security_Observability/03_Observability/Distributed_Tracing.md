# Distributed Tracing

Owner: [Your Name] (Student 4)
Diagram: Observability_Architecture.png

## 1. Purpose
Follow one purchase across checkout, payment and order, including asynchronous hops, so a problem can be located in seconds.

## 2. Standard
OpenTelemetry with W3C `traceparent` header.

## 3. Flow
`API Gateway -> Reservation Service -> Payment Service -> Message broker -> Order Service -> Notification Service`

| Step | How the trace continues |
|---|---|
| Gateway | Creates `trace_id` for the request |
| Synchronous calls | `traceparent` header passed on every call |
| Outbox events | Outbox writes `trace_id` into the message headers |
| Consumers | Read `trace_id` from headers and continue the same trace |
| Notification | Same trace ends with the customer message |

The asynchronous step is the one most teams forget. It lets you see "payment succeeded at T+0, order created at T+35 s" as one story during an Order Service outage.

## 4. Spans to create
| Span | Service |
|---|---|
| gateway.request | API Gateway |
| waiting_room.admit | Waiting room |
| reservation.redis_gate | Reservation Service |
| reservation.db_transaction | Reservation Service |
| payment.charge | Payment Service |
| outbox.publish | Outbox publisher |
| order.create | Order Service |
| notification.send | Notification Service |

Span attributes: reservation_id, payment transaction_ref (not card data), order_id, result, error_code.

## 5. Sampling
| Traffic | Sampling |
|---|---|
| Errors | 100 percent |
| Slow requests (above threshold) | 100 percent |
| Payment and order flows | 100 percent |
| SOLD_OUT and browse | About 5 percent (assumption) |

Use tail-based sampling at the collector where possible, so errors are always kept. At 500,000 requests per second, full tracing would cost more than the system it watches.

## 6. Customer support reference
The `trace_id` is returned in API responses (for example `X-Trace-Id`) so support can find the whole journey.

## 7. Using traces in incidents
| Question | Answer from the trace |
|---|---|
| Where is the latency? | Longest span |
| Did the event reach the Order Service? | `outbox.publish` exists but no `order.create` span |
| Did a retry happen? | Repeated spans with attempt number |

## 8. Privacy
Do not put personal data or secrets in span attributes.

## 9. Retention (assumption)
Traces for 7 days, error traces longer if needed.

## 10. Trade-offs
Instrumentation adds a small CPU cost and storage. Sampling and short retention keep it under control.
