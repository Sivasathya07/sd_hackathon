# Metrics



## 1. Approach
Use the four golden signals for every service (rate, errors, latency, saturation), plus business metrics for the sale.

## 2. Required metrics (from the problem statement)
| Required | Metric |
|---|---|
| Request rate | Requests per second by endpoint |
| Latency | p50, p95, p99 per endpoint |
| Error rate | 4xx and 5xx per service |
| Inventory reservation failures | SOLD_OUT, DB 0-row rejections, reservation errors |
| Payment failures | Failure, timeout and decline rates |
| Order conversion | Orders confirmed divided by reservations created |

## 3. Full metric catalogue
| Area | Metrics | Why |
|---|---|---|
| Traffic | Requests per second, WAF blocks, 429 count | Shows the surge and abuse |
| Latency | p50, p95, p99 for reservation, payment, order | The tail is what users feel |
| Errors | 4xx and 5xx per service | Separates client mistakes from our faults |
| Saturation | Gateway CPU, DB connection pool use, Redis memory and ops per second, queue depth | Early warning before collapse |
| Waiting room | Admitted, queued, rejected, queue wait time | Proves the funnel protects the core |
| Inventory | Reservations created, SOLD_OUT, active reservations, expired and released | Reservation health |
| Inventory safety | Invariant check result, Redis vs DB drift, **oversell counter (must be 0)** | Hard guarantee watched continuously |
| Payment | Success rate, failure rate, timeout rate, gateway latency, refunds | Payment health |
| Orders | Orders created, paid-without-order, order creation delay | Detects Order Service problems |
| Conversion | Confirmed orders / reservations | Business outcome |
| Messaging | Consumer lag, oldest message age, retry count, DLQ depth | Backlog early warning |
| Resilience | Circuit breaker state per dependency, reconciler heartbeat and repairs | Protection mechanisms working |
| Security | Auth failures, 401, 403, webhook signature failures | Attack detection |

## 4. Labels (dimensions)
Service, endpoint, status code, sale_id, product_id (bounded), dependency name. Avoid unbounded labels such as customer_id or trace_id in metrics, because they explode storage.

## 5. Collection
Services expose metrics, an OpenTelemetry collector or Prometheus scrapes them, Grafana shows dashboards, Alertmanager evaluates rules. Telemetry is sent off the request path, so a monitoring outage never blocks a purchase.

## 6. Sale war-room dashboard (one screen)
| Panel | Shows |
|---|---|
| Funnel | Requests, admitted, reserved, paid, confirmed |
| Stock | Available, reserved, sold, Redis counter, drift |
| Payments | Success, fail, timeout, breaker state |
| Orders and queue | Orders created, paid-without-order, consumer lag, DLQ |
| Health | p99 latency, 5xx rate, DB pool, Redis ops |

## 7. First-minute watch list
Admission rate, reservation success vs SOLD_OUT, DB pool use, p99 latency, invariant check.

## 8. Retention (assumptions)
High-resolution metrics for a few days, downsampled for 30 to 90 days.

## 9. Trade-offs
More metrics cost storage and CPU. We keep labels bounded and focus on signals that drive decisions.
