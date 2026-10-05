# Audit Logging


## 1. Purpose
Answer "who did what, when, from where, and with what result" for security, disputes, and investigations. Audit logs are separate from application logs: they are a tamper-resistant record of important actions.

## 2. What is audited
| Area | Events |
|---|---|
| Authentication | Login success and failure, MFA, token refresh, logout, lockout |
| Reservation | RESERVATION_CREATED, RESERVATION_RELEASED, SOLD_OUT, DUPLICATE_REQUEST |
| Payment | PAYMENT_INITIATED, PAYMENT_SUCCESS, PAYMENT_FAILED, PAYMENT_TIMEOUT, REFUND_ISSUED |
| Order | ORDER_CREATED, ORDER_STATE_CHANGED, ORDER_CANCELLED |
| Admin | Sale configuration, stock changes, price changes, role changes |
| Reconciler | STOCK_REPAIRED, INVARIANT_BREACH, stuck payment resolved |
| Security | Authorization denied, rate limit blocks, webhook signature failures |
| Secrets | Vault access, rotation |

## 3. Audit record fields
| Field | Example |
|---|---|
| `timestamp` | 2026-10-05T10:15:02Z (UTC) |
| `trace_id` | Same ID used across checkout, payment and order |
| `actor` | `customer:1042`, `admin:7`, `system:reconciler` |
| `action` | `REFUND_ISSUED` |
| `resource` | `payment:txn_ref`, `reservation:uuid` |
| `result` | `SUCCESS`, `DENIED`, `FAILED` |
| `source_ip` | Masked or hashed per policy |
| `before_after` | Previous and new values for repairs and admin changes |
| `reason` | Required for manual and admin actions |

Example:

```json
{
  "timestamp": "2026-10-05T10:15:02Z",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "actor": "system:reconciler",
  "action": "STOCK_REPAIRED",
  "resource": "inventory:p-100",
  "result": "SUCCESS",
  "before_after": {"redis_counter": {"before": 7, "after": 5}}
}
```

## 4. What must never be logged
Card numbers, CVV, full payment tokens, passwords, JWTs, refresh tokens, secrets, full personal data. Mask or hash where a reference is needed.

## 5. Protection of the audit log
| Control | Purpose |
|---|---|
| Append-only store | Records cannot be edited or deleted by applications |
| Separate write permission | Services can write, only a small audit role can read |
| Forwarded to SIEM | Central search and alerting |
| Integrity checks (hash chain or write-once storage) | Detect tampering |
| Retention | Longer than application logs (for example 1 year, assumption), per policy and legal needs |

## 6. How the audit log is written
Audit events are written asynchronously so a logging outage does not block a purchase. Critical events (refunds, admin changes) use a reliable path (outbox) so they are not lost.

## 7. Uses
- Investigate a disputed payment or refund.
- Detect insider misuse (unexpected stock changes or refunds).
- Prove inventory correctness: ledger and audit entries match the inventory counters.
- Security alerts from the SIEM (many denied requests, signature failures).

## 8. Link to the database audit trail
The `inventory_ledger` table records every inventory transition. The audit log records who or what triggered it.

## 9. Trade-offs
Extra storage and a little write cost, in exchange for traceability and accountability.

>
