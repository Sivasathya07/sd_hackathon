# Alerting

## 1. Rules
1. Alert on symptoms users feel and on invariants that must never break.
2. Do not alert on every CPU blip. Too many alerts means nobody reads them.
3. Every alert links to a runbook with a first action.
4. Critical alerts page on-call. Warnings go to chat.

## 2. Alert table (thresholds are assumptions)
| Alert | Condition | Severity | First action |
|---|---|---|---|
| Inventory invariant breach | `available + reserved + sold` not equal to `total`, or oversell counter above 0 | **Critical, page** | Freeze the SKU, fail closed, check the ledger |
| Redis vs DB drift persists | Different for more than 1 minute after re-read | Warning | Check reconciler, rebuild counter |
| Payment failure spike | Failure rate above 10% for 5 minutes (5% is normal declines in the test case) | **Critical** | Check gateway status and breaker |
| Gateway latency | p99 above 2 s for 2 minutes | Warning | Expect breaker to open, prepare user message |
| Paid without order | SUCCESS payments with no order for more than 5 minutes | **Critical** | Check Order Service, confirm re-publish, watch refund cap |
| Queue backlog | Consumer lag above 1,000 or oldest message above 60 s | Warning, Critical after 5 minutes | Scale consumers, check Order Service |
| DLQ not empty | DLQ depth above 0 | Warning | Inspect, fix cause, replay |
| Circuit breaker open | Any breaker OPEN for more than 1 minute | Warning | Check the dependency |
| Reservation latency | p99 above 500 ms for 2 minutes | Warning | Check DB pool, Redis, waiting room cap |
| Error rate | 5xx above 2% for 3 minutes | Critical | Check recent deploy and dependencies |
| Reconciler heartbeat missing | No heartbeat for 2 minutes | Warning | Restart, check leader election |
| Sweeper not releasing | RESERVED past expires_at count growing | Warning | Restart sweeper |
| DB failover or replication lag | Primary unhealthy, or lag above 5 s | **Critical** | Confirm promotion, watch 503 rate |
| Waiting room saturation | Queue above capacity for 2 minutes | Warning | Review admission rate |
| Auth attack | 401 or 403 spike, webhook signature failures | Warning | Review WAF, block sources |
| Monitoring stack down | Telemetry heartbeat missing | Warning | Restore monitoring, sale continues |

## 3. Failure matrix to alert mapping
| Failure | Detected by |
|---|---|
| Redis down | Drift alert, Redis latency and error metrics |
| DB primary down | DB failover alert, 5xx rate |
| Gateway down | Payment failure spike, gateway latency, breaker open |
| Order Service down | Paid without order, queue backlog |
| Broker down | Publish failures, consumer lag |
| Instance crash | Load balancer health, error rate |
| Reconciler stopped | Heartbeat alert |

## 4. Routing
| Severity | Channel |
|---|---|
| Critical | Pager to on-call, then escalate if not acknowledged in 5 minutes |
| Warning | Team chat channel |
| Info | Dashboard only |

## 5. Runbook template
1. What the alert means
2. Customer impact
3. Check these dashboards or traces
4. First action
5. Escalation contact
6. How to confirm recovery

## 6. Avoiding alert fatigue
Group related alerts, add durations (for example "for 5 minutes"), and review noisy alerts after the sale.

## 7. Trade-offs
Tight thresholds catch problems early but may cause false alarms. We start with these values and tune after load testing.


