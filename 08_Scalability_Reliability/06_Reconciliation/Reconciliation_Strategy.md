# Reconciliation Strategy



## 1. Purpose
The reconciler is the safety net behind the two-store design (Redis and PostgreSQL) and the payment and order workflow.
**PostgreSQL is the source of truth. The reconciler repairs everything else to match it.**

## 2. Schedule and leadership
- Inventory checks every 30 s (assumption). Payment and order checks on a slower cycle.
- Only one instance runs at a time (leader election with a DB advisory lock or lease).
- Heartbeat metric is published. Alert if missing for 2 minutes.

## 3. Checks
| # | Check | Compares | Mismatch means | Repair |
|---|---|---|---|---|
| 1 | Invariant | `available + reserved + sold` vs `total` | A real bug or rogue writer | **No auto-repair.** Critical alert, freeze the SKU, fail closed |
| 2 | Ledger | Sum of `inventory_ledger` vs inventory counters | Transition without audit entry or the reverse | **No auto-repair.** Critical alert, human investigation |
| 3 | Redis vs DB | Redis counter vs `available_quantity` | Crash between Redis decrement and DB commit, or Redis data loss | Overwrite Redis with DB value |
| 4 | Stuck payments | PENDING payments older than 2 minutes vs gateway status by transaction_ref | Crash after charge, before commit | Update to the gateway's answer |
| 5 | Paid without order | SUCCESS payments with no order after 5 minutes | Order Service outage or lost event | Re-publish, refund after 30 minute cap |
| 6 | Expired reservations | RESERVED past expires_at | Sweeper missed or crashed | Release with conditional update |

## 4. Redis vs DB drift handling
1. If different, wait 5 seconds and re-read, to skip in-flight requests (between Redis decrement and DB commit).
2. If still different:
   - Redis lower than DB: Redis too strict, sales being lost. Set Redis to the DB value and log the repair.
   - Redis higher than DB: Redis too generous, but the DB still rejects. Set Redis to the DB value, log the repair, raise a warning.

## 5. Stuck payment handling
| Gateway status | Action |
|---|---|
| Success | Mark payment SUCCESS, emit PaymentConfirmed |
| Failed or not found | Mark FAILED, release reservation |
| Unknown or timeout | Keep PENDING, retry next cycle |

## 6. Design rules
1. **One-way repair.** Redis is repaired to match PostgreSQL, never the reverse.
2. **Expected drift is auto-repaired. Unexpected corruption is not.** A broken invariant or ledger mismatch is never silently fixed, because that could hide a bug or fraud.
3. **Every repair is idempotent.** Releases use conditional updates, refunds use a derived refund_ref, re-published events are deduplicated. Running it twice does no harm.
4. **Every repair is audited** with before and after values (see 09_Security_Observability audit logging).
5. **Correctness does not depend on the reconciler.** The DB conditional update still prevents oversell. The reconciler restores efficiency and cleans stuck states.

## 7. Outputs
- Reconciliation report and metrics per cycle.
- Metrics: drift count, repairs, stuck payments, paid-without-order count, heartbeat.
- Alerts: invariant breach (critical), drift persists more than 1 minute (warning), heartbeat missing (warning).

## 8. Trade-offs
- Allows brief Redis and DB drift instead of two-phase commit, which is slow and fragile under flash-sale load.
- Adds one more component to run and monitor.
