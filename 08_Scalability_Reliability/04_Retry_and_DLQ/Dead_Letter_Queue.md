# Dead Letter Queue (DLQ) and Idempotent Consumers

## 1. Purpose
A message that keeps failing must not block healthy messages, and must never be silently dropped.
Every message ends in exactly one of two places: **success** or the **DLQ**.

## 2. When a message goes to the DLQ
| Case | Action |
|---|---|
| Retries exhausted (8 attempts) | Move to DLQ |
| Permanent error (validation failure, malformed message) | Move to DLQ immediately |
| Poison message that crashes the consumer repeatedly | Move to DLQ after the attempt limit |

## 3. DLQ handling
1. Message moves to the DLQ with metadata: original topic, event_id, attempt count, last error, trace_id, timestamps.
2. Alert fires when DLQ depth is above 0 (warning).
3. An engineer inspects the message using the trace_id and fixes the root cause.
4. A replay tool re-publishes the message to the main queue.
5. Replay is safe because consumers are idempotent.
6. Retention: 14 days (assumption), long enough to investigate and replay.

## 4. DLQ vs reconciler
For paid-but-unfulfilled orders the reconciler also scans for "payment SUCCESS with no order", so a customer never depends on a human noticing the DLQ.

## 5. Idempotent consumer
Delivery is at-least-once, so the same message can arrive many times (redelivery, retry, DLQ replay, reconciler re-publish).

Flow:
1. Receive message with event_id.
2. BEGIN transaction.
3. INSERT event_id into `processed_events`. A unique violation means duplicate, so ROLLBACK, ack and skip.
4. Apply the business change (for example create the order).
5. INSERT the outbox event (for example OrderCreated).
6. COMMIT, then ack.

```sql
CREATE TABLE processed_events (
  event_id      UUID        NOT NULL,
  consumer_name VARCHAR(50) NOT NULL,
  processed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (event_id, consumer_name)
);
```

The key includes `consumer_name` because different services (Order, Notification) may each consume the same event once.

## 6. Three layers of duplicate protection
| Layer | Mechanism | Stops |
|---|---|---|
| 1. Event dedupe | UNIQUE(event_id, consumer_name) | Same message delivered twice |
| 2. Business dedupe | UNIQUE(payment_id) on orders | Different events for the same payment (reconciler re-publish) |
| 3. State check | Update only if current state allows (`WHERE status = 'CREATED'`) | Out-of-order or stale updates |

## 7. Consumer rules
1. Insert event_id and apply the business change in **one transaction**.
2. Ack only after commit.
3. A duplicate is a success, not an error. Do not send it to the DLQ.
4. External side effects need their own key (refund_ref, notification_key).
5. Tolerate out-of-order delivery.

## 8. Cleanup
A job deletes `processed_events` rows older than the retention window (14 days, matching DLQ retention). UNIQUE(payment_id) still protects after cleanup.

## 9. Monitoring
Metrics: DLQ depth, oldest DLQ message age, retry count, consumer lag. Alerts: DLQ not empty, queue backlog.
