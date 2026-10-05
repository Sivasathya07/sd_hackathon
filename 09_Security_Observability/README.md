# 09 - Security and Observability Design


## Purpose
How SALESTORM protects customers, payments and inventory from abuse and attack, and how the team detects problems quickly. Every failure in 08_Scalability_Reliability has a detector defined here.

## Structure
| Folder | Files | Topic |
|---|---|---|
| 01_Security | Authentication_and_Authorization.md, HTTPS_and_Transport_Security.md, Rate_Limiting.md, Input_Validation.md, Payment_Data_Security.md, Secrets_Management.md, Security_Architecture.png | Security controls and trust boundaries |
| 02_Audit_Logging | Audit_Logging.md | Tamper-resistant record of important actions |
| 03_Observability | Metrics.md, Structured_Logging.md, Distributed_Tracing.md, Alerting.md, Observability_Architecture.png | Metrics, logs, traces, alerts |

## Security summary
| Requirement | Control |
|---|---|
| Authentication | Identity provider, short-lived JWT (15 min), verified at the API Gateway |
| Authorization | Role checks at the Gateway, ownership checks in each service |
| HTTPS | TLS 1.3 to the edge, mTLS between services |
| Rate limiting and abuse | WAF, per-IP and per-user limits, bot challenge, one active reservation per customer |
| Input validation | Schema validation at the Gateway, re-validation in services, parameterised queries |
| Payment data | Card data never reaches our servers. Only token, last 4 digits and transaction_ref are stored |
| Audit logging | Append-only log forwarded to SIEM |
| Secrets | Vault with rotation, nothing in code |

## Observability summary
| Area | Design |
|---|---|
| Metrics | Request rate, latency (p50, p95, p99), error rate, reservation failures, payment failures, order conversion, saturation, queue lag, DLQ depth, breaker state |
| Logs | Structured JSON with trace_id on every critical event |
| Tracing | One trace_id from Gateway through reservation, payment, order and notification, including asynchronous hops |
| Alerts | Inventory invariant breach, payment failure spike, paid-without-order, queue backlog, DLQ not empty, DB failover |

## Reading order
1. Authentication_and_Authorization.md
2. HTTPS_and_Transport_Security.md and Rate_Limiting.md
3. Input_Validation.md, Payment_Data_Security.md, Secrets_Management.md
4. Audit_Logging.md
5. Metrics.md, Structured_Logging.md, Distributed_Tracing.md
6. Alerting.md

## Assumptions
JWT lifetime, rotation periods, rate limits, sampling rate, retention periods and alert thresholds are design choices, not measured values. Tune them with load testing and the team security policy.
