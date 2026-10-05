# HTTPS and Transport Security


## 1. Rule
No plain-text traffic anywhere. Every hop is encrypted and, inside the system, mutually authenticated.

## 2. Hop-by-hop
| Hop | Protection |
|---|---|
| Customer to CDN/WAF | HTTPS, TLS 1.3 (TLS 1.2 minimum fallback), HSTS enabled |
| CDN to load balancer | TLS |
| Load balancer to API Gateway | TLS |
| Gateway to services | mTLS with service identity |
| Services to PostgreSQL | TLS, encrypted at rest |
| Services to Redis | TLS and AUTH, private network only |
| Services to message broker | mTLS and per-topic ACL |
| Payment Service to gateway | HTTPS with API key from the vault |
| Payment gateway webhook to us | HTTPS plus signature verification |

## 3. Network zones
| Zone | Contents | Access |
|---|---|---|
| Zone 0 | Internet (untrusted) | n/a |
| Zone 1 | CDN, WAF, load balancer | Public entry |
| Zone 2 | API Gateway and application services | Private, no direct internet |
| Zone 3 | PostgreSQL, Redis, broker, audit store | No internet access, only from Zone 2 |

Security groups and network policies allow only the required ports between zones.

## 4. Certificates
- Public certificates managed and auto-renewed at the edge.
- Internal certificates are short-lived and rotated automatically.
- Private keys never stored in code or images.

## 5. Headers and settings
- HSTS, secure and HttpOnly cookies, SameSite where cookies are used.
- Strict CORS allow-list.
- Disable weak ciphers and old protocols.

## 6. Data at rest
- Database and backups encrypted at rest.
- Audit log store is append-only.
- Sensitive fields masked in logs.

## 7. Zero trust inside the network
Being inside the private network is not enough. Every internal call is authenticated.

## 8. Trade-offs
Certificate rotation and mTLS add operational complexity and small latency. A service mesh automates this.

