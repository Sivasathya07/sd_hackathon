# Retry Strategy

## 1. Purpose
Recover automatically from transient failures without hammering a struggling dependency and without creating duplicate effects.

## 2. Timeouts (assumptions)
| Call | Timeout | Reason |
|---|---|---|
| Reservation Service to Redis | 50 ms | Redis is in-memory and fast |
| Reservation Service to PostgreSQL | 2 s | Short transactions only |
| Payment Service to gateway | 3 s | External dependency |
| Consumer to Order Service | 1 s | Internal service |
| Client to API (overall) | 10 s | User-facing limit |

Never wait forever. Every network call has a timeout.

## 3. Retry policy for asynchronous messages
| Parameter | Value | Why |
|---|---|---|
| Max attempts | 8 | Covers the 30 s Order Service outage with margin |
| Base delay | 1 s | Fast first retry for brief blips |
| Backoff | Exponential (x2): 1, 2, 4, 8, 16, 30, 30 s | Gives the dependency room to recover |
| Max delay cap | 30 s | Keeps recovery time bounded |
| Jitter | Full jitter (random between 0 and the delay) | Prevents synchronised retries (retry storm) |
| Total window | About 2 minutes worst case | Long enough for transient faults, short enough to surface real problems |

## 4. What is retried
| Retryable (likely to succeed later) | Not retryable (go straight to DLQ) |
|---|---|
| Timeout | Validation failure |
| 5xx response | Malformed message |
| Connection error | 4xx business rejection |
| Circuit breaker open (pause, do not burn attempts) | Unknown event type |

## 5. Synchronous calls
| Call | Retry rule |
|---|---|
| Redis gate | No retry. Fall back to DB path. |
| DB reservation transaction | At most 1 immediate retry for serialization or deadlock errors. Otherwise 503. |
| Gateway charge | **No blind retry.** Outcome is unknown after a timeout. Query status by transaction_ref instead. |
| Gateway status query | Safe to retry with backoff (read-only). |
| Client to API | Client retries with the same Idempotency-Key and jittered backoff. |

## 6. Safe retries need idempotency
Retries mean the same request or message can arrive more than once. Each hop has its own key:
| Boundary | Key |
|---|---|
| Client to API | Idempotency-Key |
| Payment to gateway | transaction_ref |
| Refund | refund_ref (derived from transaction_ref) |
| Events | event_id plus processed_events table |
| Order creation | UNIQUE(payment_id) |

## 7. Retry budget
Retries must not multiply load. Cap retries at a fraction of normal traffic (for example 10 percent) so a failing dependency is not overwhelmed. When the budget is spent, fail fast or send to the DLQ.

## 8. Interaction with the circuit breaker
While the breaker is OPEN, consumers pause and keep messages in the queue. When it goes HALF-OPEN, 3 probes are sent first. This prevents a recovery stampede.

## 9. Trade-offs
- Backoff delays recovery slightly in exchange for protecting the dependency.
- Retries give at-least-once delivery, so every consumer must be idempotent.
- Message order is not guaranteed across retries, so consumers check current state before applying a transition.

> 
