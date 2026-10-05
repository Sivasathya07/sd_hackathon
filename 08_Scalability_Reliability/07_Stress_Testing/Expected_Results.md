# Expected Results

## 1. Base case outcome (S1, S2, S4)
| Metric | Expected | Actual |
|---|---|---|
| Requests sent | 10,000 (including about 200 duplicates) | |
| Reservations created | 100 (never more) | |
| SOLD_OUT responses | About 9,900 (minus duplicates and rate-limited) | |
| Duplicate requests returning the same reservation | About 200 | |
| Payments succeeded | About 95 | |
| Payments failed | About 5, reservations released | |
| Units released back to available | About 5 | |
| Final sold (if released units are re-sold via waitlist) | 100 | |
| Final sold (no re-sale) | About 95, with 5 available | |
| Oversold units | **0** | |
| available + reserved + sold = total | **True** | |
| Negative stock at any time | **Never** | |

## 2. Per scenario expectations
| Scenario | Expected result | Actual |
|---|---|---|
| S3 Last unit | 1 winner (201), 1 loser (409), available = 0 | |
| S5 Expiry | Overdue reservations released once, available +1 per release | |
| S6 Payment timeout | No double charge, resolved via transaction_ref | |
| S7 Order Service down 30 s | All paid customers get exactly one order, no DLQ entries, delay of seconds | |
| S8 Duplicate messages | One order per payment | |
| S9 Poison message | DLQ entry and alert, other messages unaffected | |
| S10 Redis down | Fallback to DB, no oversell | |
| S11 Crash after Redis decrement | Drift repaired within about 30 s | |
| S12 DB failure | 503 for reservations, recovery within RTO target, no lost committed data | |
| S13 Gateway outage | Breaker opens, no duplicate charge, reconciled afterwards | |
| S14 50x traffic | DB and Redis load near the base case, no oversell | |
| S15 Bot | 429 and one hold per customer | |

## 3. Targets (assumptions, not guarantees)
| Target | Value |
|---|---|
| Reservation p99 latency | Under 500 ms |
| Database failover RTO | Under 60 s |
| Database failover RPO | Zero (synchronous replication) |
| Order Service outage recovery | Orders created within minutes of recovery |
| Reconciler drift correction | About 30 s |

## 4. Strict guarantees vs targets
| Strict guarantees (must always hold) | Targets (aim for, may vary) |
|---|---|
| Sold never exceeds 100 | Latency numbers |
| No duplicate reservation, charge or order | RTO and recovery times |
| Every paid customer gets an order or refund | Throughput |
| Invariant available + reserved + sold = total | Waiting time in the queue |

## 5. How to record results
1. Run the simulation or load test for each scenario.
2. Fill the Actual column.
3. Save a screenshot or chart of the outcome as `Test_Results.png` in this folder.
4. If a result differs from expectation, write what you changed in the design (this shows the validation loop to the jury).

## 6. AI-assisted simulation note
If AI generated the simulation script, add the tool and purpose to the AI Usage Note and explain how you reviewed and tested it.
