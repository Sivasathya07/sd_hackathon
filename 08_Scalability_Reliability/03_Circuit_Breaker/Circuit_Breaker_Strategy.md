# Circuit Breaker Strategy

## 1. Purpose
Without a breaker, every request waits for a dead dependency, threads fill up, and the whole service collapses.
A breaker fails fast, protects the caller, and gives the dependency room to recover.

## 2. States
| State | Behaviour |
|---|---|
| CLOSED | All calls pass through. Failures are tracked in a 10 second sliding window. |
| OPEN | No calls are made. A fallback is returned immediately. Wait for the cool-down. |
| HALF-OPEN | Only 3 probe calls are allowed. Everything else fails fast. |

Transitions:
- CLOSED -> OPEN: failure rate at or above 50 percent in the window (minimum 20 calls), or slow calls at or above 50 percent.
- OPEN -> HALF-OPEN: cool-down elapsed.
- HALF-OPEN -> CLOSED: 3 of 3 probes succeed.
- HALF-OPEN -> OPEN: any probe fails (restart cool-down).

## 3. Thresholds (assumptions)
| Parameter | Payment gateway | Order Service | Why |
|---|---|---|---|
| Call timeout | 3 s | 1 s | Gateway is external and slower |
| Failure-rate threshold | 50% | 50% | Opens only on sustained failure |
| Sliding window | 10 s | 10 s | Reacts quickly during a flash sale |
| Minimum calls | 20 | 20 | Avoid opening on 1 failure out of 2 calls |
| Slow-call threshold | above 2 s | above 500 ms | Slowness exhausts threads like failure does |
| Open duration | 30 s | 15 s | Time for the dependency to recover |
| Half-open probes | 3 | 3 | Test recovery without a flood |
| Counts as failure | Timeout, 5xx, connection error | Timeout, 5xx, connection error | **4xx is not a failure** |

A declined card (4xx) is a valid answer from a healthy gateway, so it does not count.

## 4. Behaviour per dependency
| State | Payment gateway | Order Service |
|---|---|---|
| CLOSED | Normal payment calls | Normal order creation |
| OPEN | No gateway calls. Reservation held (TTL extended within a 15 minute cap). User sees "payment pending". | No calls. Payment stays CONFIRMED and the event stays in the outbox or queue. |
| HALF-OPEN | 3 probe calls, status query only. Pending payments are checked by transaction_ref, never re-charged. | 3 events sent. If they succeed, the backlog drains gradually. |

## 5. Interaction with other mechanisms
- **Retries:** retries use backoff and jitter. While the breaker is open, consumers pause instead of burning attempts, so a short outage does not push messages to the DLQ.
- **Reservation TTL:** reservations in PAYMENT_PENDING are not swept while the breaker is open, up to the hard cap.
- **Double charge safety:** when the breaker opens no new charge calls are made. For in-flight payments with unknown outcome, the reconciler queries the gateway by transaction_ref.

## 6. Where it lives
In the calling service, per dependency, with in-memory state per instance (for example a Resilience4j-style library).
Trade-off: each instance learns of failure independently (slightly slower), but there is no shared-state dependency.

