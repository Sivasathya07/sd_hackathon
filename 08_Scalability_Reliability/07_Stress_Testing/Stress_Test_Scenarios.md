# Stress Test Scenarios

Tools: Locust, JMeter or a Python simulation (AI-assisted use must be noted in the AI usage note).
These scenarios are written for design validation. Record real results in Expected_Results.md and Test_Results.png only after you run them.

## S1. Core flash sale (10,000 vs 100)
- **Setup:** 100 units, 10,000 concurrent Buy Now requests.
- **Steps:** Fire all requests together, then count outcomes.
- **Checks:** reservations created, SOLD_OUT responses, final sold count, invariant.
- **Pass:** reservations never exceed 100, sold never exceeds 100, available never negative, invariant holds.

## S2. Duplicate requests (2 percent)
- **Setup:** About 200 requests reuse an earlier Idempotency-Key.
- **Pass:** each duplicate returns the same reservation_id, no extra stock consumed, no extra row.

## S3. Last-unit race
- **Setup:** 1 unit left, 2 simultaneous requests from different customers.
- **Pass:** exactly one 201, the other 409 SOLD_OUT, available = 0, never negative.

## S4. Payment failure (5 percent)
- **Setup:** 100 reservations, 5 payments fail.
- **Pass:** 5 reservations RELEASED, reserved -5, available +5, released units can be re-reserved, invariant holds.

## S5. Reservation expiry
- **Setup:** Reservation TTL shortened (for example 10 s for the test). Some customers never pay.
- **Pass:** sweeper releases overdue reservations once, no double release, available increases exactly once per release.

## S6. Payment timeout (unknown outcome)
- **Setup:** Gateway stub delays beyond 3 s, then reports the charge succeeded.
- **Pass:** no second charge, payment stays PENDING then moves to SUCCESS via reconciliation by transaction_ref.

## S7. Order Service down for 30 seconds
- **Setup:** Stop the Order Service for 30 s after payments succeed.
- **Pass:** PaymentConfirmed events retained, retries with backoff and jitter, breaker opens then closes, every paid customer gets exactly one order, no DLQ entries for this short outage, inventory unaffected.

## S8. Duplicate message delivery
- **Setup:** Deliver the same PaymentConfirmed event several times, including two consumers at once.
- **Pass:** exactly one order per payment, duplicates acked and skipped.

## S9. Poison message
- **Setup:** Publish a malformed event.
- **Pass:** goes to the DLQ without blocking others, alert fires, replay after fix succeeds.

## S10. Redis failure
- **Setup:** Kill Redis during the sale.
- **Pass:** requests fall back to the DB path with tighter admission, no oversell, counter rebuilt from DB after recovery.

## S11. Crash between Redis decrement and DB commit
- **Setup:** Inject a crash after the Lua decrement.
- **Pass:** reconciler detects drift after the 5 s re-read and corrects Redis to the DB value. No oversell.

## S12. Database primary failure
- **Setup:** Stop the primary during the sale.
- **Pass:** reservations return 503 with Retry-After, no Redis-only sales, replica promoted, retries with same Idempotency-Key succeed, invariant verified after recovery.

## S13. Payment gateway outage
- **Setup:** Gateway returns 5xx or times out for several minutes.
- **Pass:** breaker opens, reservations held within the cap, no duplicate charges, pending payments reconciled by transaction_ref, late success after release is re-reserved or auto-refunded.

## S14. 50x traffic
- **Setup:** Raise request rate to 50 times normal (500,000 requests per second target, scaled to your test environment).
- **Pass:** waiting room caps admission, Redis and DB load stay near the same as the base case, gateway latency stays within target, no oversell. Record where the first bottleneck appears.

## S15. Bot behaviour
- **Setup:** One account sends many different Idempotency-Keys.
- **Pass:** rate limit returns 429 and the unique active-reservation index prevents more than one hold per customer per sale.

## Scenario to guarantee map
| Guarantee | Scenarios |
|---|---|
| No oversell | S1, S3, S10, S11, S12, S14 |
| No duplicate reservation, charge or order | S2, S6, S8, S15 |
| Reservation release correctness | S4, S5 |
| Paid customer gets an order or refund | S7, S9, S13 |
| Graceful degradation | S10, S12, S13, S14 |
