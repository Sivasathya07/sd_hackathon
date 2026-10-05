# Rate Limiting and Abuse Protection
## 1. Why
A flash sale attracts bots, scripts, replay attacks and floods. We must protect real customers and inventory.

## 2. Layers
| Layer | Control | Stops |
|---|---|---|
| CDN/WAF | DDoS filtering, bot rules, per-IP rate limit, challenge on the sale page | Floods, scripts |
| API Gateway | Per-user and per-endpoint rate limit | Rapid repeated clicks, one account hammering |
| API Gateway | Idempotency-Key required on write calls | Duplicate and replayed requests |
| Waiting room | Admission cap (token bucket) | Overload of the reservation path |
| Database | Unique partial index: one active reservation per customer per sale | A bot that passes everything else |

## 3. Example limits (assumptions)
| Scope | Limit |
|---|---|
| Per IP at the WAF | Tuned to shared networks (colleges, offices share IPs) so real users are not blocked |
| Per user, reservation endpoint | 5 requests per 10 seconds |
| Per user, login | 5 failures then temporary lockout |
| Per user, payment endpoint | 3 requests per minute |

## 4. Algorithm
Token bucket or sliding window counters, kept in Redis so all gateway instances share limits. Fall back to a local limit if Redis is unavailable (fail safe, slightly looser).

## 5. Response
429 Too Many Requests with a Retry-After header. Clients use exponential backoff with jitter.

## 6. Bot defence
- Challenge before the waiting room (CAPTCHA or proof of work).
- Account age and verification checks for high-demand sales.
- Device and behaviour signals at the WAF.
- Per-account purchase limit (one reservation).

Honest note: no bot defence is perfect. The database unique index is the hard stop.

## 7. Replay protection
Short JWT lifetime, TLS, and Idempotency-Key. A replayed request returns the same result and creates no new effect.

## 8. Monitoring
Metrics: 429 count, WAF blocks, bot challenge failures, admission rate. Alert on unusual spikes.

## 9. Trade-offs
- Strict limits may block some honest users (shared IPs). Per-user limits are preferred over per-IP where possible.
- Rate limiting state in Redis adds a dependency, so a local fallback exists.

