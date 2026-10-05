# Structured Logging

## 1. Principle
Logs are JSON, one event per line, never free text. Every critical business event shares core fields so we can search by `trace_id` or `reservation_id`.

## 2. Standard format
```json
{
  "timestamp": "2026-10-05T10:15:02.123Z",
  "level": "INFO",
  "service": "reservation-service",
  "instance": "rs-7f9c",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "event": "RESERVATION_CREATED",
  "customer_id": "c-1042",
  "reservation_id": "res-uuid",
  "product_id": "p-100",
  "idempotency_key": "9b2e...",
  "result": "SUCCESS",
  "latency_ms": 18
}
```

## 3. Core fields
| Field | Purpose |
|---|---|
| timestamp (UTC) | Ordering |
| level | DEBUG, INFO, WARN, ERROR |
| service, instance | Where it happened |
| trace_id, span_id | Link to distributed trace |
| event | Stable event name |
| result | SUCCESS, FAILED, DENIED, DUPLICATE |
| latency_ms | Performance |
| error_code | Stable code on failure |

## 4. Critical business events
| Event | Logged by |
|---|---|
| RESERVATION_CREATED, SOLD_OUT, RESERVATION_RELEASED, DUPLICATE_REQUEST | Reservation Service |
| PAYMENT_INITIATED, PAYMENT_SUCCESS, PAYMENT_FAILED, PAYMENT_TIMEOUT, REFUND_ISSUED | Payment Service |
| ORDER_CREATED, ORDER_CREATION_RETRY, ORDER_STATE_CHANGED | Order Service |
| BREAKER_OPENED, BREAKER_CLOSED, MESSAGE_DEAD_LETTERED | Resilience layer |
| STOCK_REPAIRED, INVARIANT_BREACH | Reconciler |

## 5. Levels
| Level | Use |
|---|---|
| ERROR | Operation failed and needs attention |
| WARN | Degraded but handled (retry, breaker open) |
| INFO | Business events above |
| DEBUG | Disabled in production by default |

At 500,000 requests per second, per-request INFO logs for SOLD_OUT would be too costly. Log rejected-at-gateway requests as counters, and sample detailed logs (for example 1 to 5 percent), keeping 100 percent of errors, payments and state changes.

## 6. Never log
Card data, full payment tokens, passwords, JWTs, secrets, full personal data. The collector masks sensitive fields as a second safety layer.

## 7. Pipeline
Service writes JSON to stdout, an agent ships logs to the collector, then to the log store (Loki or Elasticsearch). Logging is asynchronous and buffered locally for short outages.

## 8. Retention (assumptions)
Application logs 30 days. Audit logs longer (see Audit_Logging.md).

## 9. Correlation
The same `trace_id` appears in logs, traces, audit entries and the customer support reference, so one search shows the full story of a purchase.

## 10. Trade-offs
Structured logs are bigger than plain text. Sampling and retention limits control cost.
