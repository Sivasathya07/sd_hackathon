# High Traffic Scaling

## 1. Traffic assumptions
| Scenario | Load | Source |
|---|---|---|
| Normal traffic | About 10,000 requests per second | Problem statement |
| Flash-sale peak | Up to 500,000 requests per second (50x) | Problem statement |
| Purchase attempts | 10,000 concurrent for 100 units | Practical test case |

## 2. Key argument
Inventory load depends on **stock**, not on **traffic**. The funnel below keeps Redis and the database load nearly constant, even at 50x.

## 3. Traffic funnel (10,000 requests)
| Layer | Role | Requests surviving |
|---|---|---|
| CDN and WAF | Serve static content, drop bots, per-IP rate limit | Bots removed |
| Load balancer | Spread traffic over stateless gateways | n/a |
| API Gateway | JWT, per-user rate limit, Idempotency-Key check, local sold-out flag | Abusive requests get 429 |
| Waiting room | Admits only a few times the stock (about 300) | About 9,700 queued or rejected cheaply |
| Reservation Service | Stateless, calls the Redis gate | About 300 |
| Redis Lua gate | Atomic check and decrement | About 100 pass |
| PostgreSQL | Authoritative conditional update | About 100 writes |

## 4. How each tier scales
| Component | Scaling method | State |
|---|---|---|
| CDN | Edge caches, absorbs page and asset traffic | Static |
| Load balancer | Managed, multi-zone | None |
| API Gateway | Horizontal, autoscale on CPU and request rate | Stateless (JWT carries identity) |
| Reservation Service | Horizontal | Stateless, state is in Redis and DB |
| Waiting room | Token bucket, counters in Redis | Small counters |
| Redis | Primary with replica, sharded by SKU if many products | Counter plus idempotency keys |
| PostgreSQL | Single primary for inventory writes, read replicas for reads | Source of truth |
| Message broker | Partitioned topics, more consumers | Durable |

## 5. What changes at 50x (500,000 requests per second)
1. CDN and edge serve the sale page. Product reads come from cache.
2. The sold-out flag is cached locally on each gateway instance, so sold-out requests are rejected in microseconds with no network hop.
3. The waiting room caps admission, so Redis and DB load stay about the same.
4. Gateway and reservation instances scale out horizontally and are stateless.
5. If a single Redis key becomes hot (many SKUs or large stock), shard stock into N buckets (for example 10 buckets of 10 units) with an overflow check. At 100 units this is unnecessary, but it is the documented next step.
6. Telemetry is sampled (see 09_Security_Observability) so monitoring cost does not exceed system cost.

## 6. Bottlenecks and mitigation
| Bottleneck | Why | Mitigation |
|---|---|---|
| Hot inventory row | One row serialises updates (about 1,000 to 3,000 per second) | Redis gate and waiting room shield it |
| Redis hot key | One key per SKU | Stock bucketing, replica for failover |
| DB connection pool | Many concurrent transactions | Only about 100 transactions reach the DB, pool sizing, timeouts |
| Payment gateway | External, rate limited | Circuit breaker, queueing, hold reservation |
| Message broker lag | Spikes after outages | Partitioning, autoscaled consumers, backlog alert |
| Gateway CPU (TLS and JWT) | Highest request volume | Horizontal scaling, TLS offload at the load balancer |

## 7. Trade-offs
- The waiting room adds a little latency for most users in exchange for system survival.
- Local sold-out flags can be briefly stale. We accept a late rejection rather than a network hop per request.
- Single-primary writes limit raw write scale, but inventory writes are tiny (about 100 per product), so this is acceptable.

