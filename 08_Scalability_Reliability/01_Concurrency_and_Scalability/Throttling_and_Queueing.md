# Throttling and Queueing

## 1. Summary
| Action | Where | Why | Response to client |
|---|---|---|---|
| **Throttled** | CDN/WAF (per IP), API Gateway (per user and endpoint) | Stop bots, scripts and abuse | 429 Too Many Requests with Retry-After |
| **Queued** | Waiting room | Keep inventory load constant | 202 Queued with position or Retry-After |
| **Rejected** | Gateway local flag, Redis gate, DB | Stock is zero | 409 SOLD_OUT |
| **Shed** | Any overloaded service | Protect the core under saturation | 503 with Retry-After |

## 2. Request path
Customer -> CDN/WAF -> Load balancer -> API Gateway -> Waiting room -> Reservation Service -> Redis gate -> PostgreSQL

## 3. Throttling rules 
| Layer | Rule | Purpose |
|---|---|---|
| WAF | Per-IP rate limit and bot rules | Block scripts and floods |
| Gateway | Per-user limit on the reservation endpoint (for example 5 requests per 10 seconds) | Stop rapid repeated clicks |
| Gateway | Reject requests with no Idempotency-Key | Force safe retries |
| Database | One active reservation per customer per sale (unique partial index) | Hard stop against bots with many requests |

Honest note on bots: no bot defence is perfect. The unique active-reservation index is the hard stop, so even a bot that passes the WAF cannot hold more than one reservation per account.

## 4. Waiting room design
- Algorithm: token bucket with admission rate and burst size tuned to stock. Example: admit about 300 requests for 100 units.
- Counters live in Redis. The admission decision is cheap and does not touch the database.
- Admitted requests carry a short-lived admission token so they cannot be replayed.
- Not admitted: the client receives 202 (queued) or 429 with Retry-After and polls or retries with jitter.
- As reservations are released (payment failure or timeout), the waiting room admits more requests, so freed units are not wasted.

## 5. Queueing vs rejecting
| Situation | Behaviour |
|---|---|
| Stock available, admission cap reached | Queue (wait briefly) |
| Stock zero but reservations still pending | Reject with SOLD_OUT, UI shows "Sold out, join waitlist" |
| Stock zero and nothing pending | Reject with SOLD_OUT, final |
| System overloaded | Shed load with 503, protect checkout and payment first |

"Sold out" is not permanent while reservations are pending, because unpaid reservations return units.

## 6. Priority under overload
1. Payment and order completion for existing reservations (money already involved)
2. New reservations
3. Browsing and product reads (cached)

## 7. Duplicate handling inside the funnel
- Duplicate requests (about 2 percent, around 200) carry the same Idempotency-Key.
- They return the same response and consume no stock and no extra admission slot.

## 8. Client behaviour
- Exponential backoff with jitter on 429 and 503 so clients do not retry in lockstep.
- Always reuse the same Idempotency-Key when retrying the same purchase attempt.

## 9. Trade-offs
- Some honest customers wait or are rejected, in exchange for the system staying up.
- Admission caps may be slightly conservative, which can leave units unsold for a short time until freed units are re-admitted.

>
