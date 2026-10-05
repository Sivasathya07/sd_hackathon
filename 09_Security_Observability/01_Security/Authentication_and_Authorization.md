# Authentication and Authorization


## 1. Goals
- Only real, logged-in customers can reserve and pay.
- A customer can only see and change their own data.
- Services only accept calls from services that are allowed to call them.

## 2. Authentication (who are you?)
| Item | Design |
|---|---|
| Identity provider | Dedicated identity service handles login, password hashing and MFA |
| Token | Short-lived signed JWT (15 minutes, assumption) plus refresh token |
| Verification | API Gateway verifies signature, expiry, issuer and audience using the identity provider's public key |
| MFA | Required for admin and support accounts, step-up for sensitive actions |
| Refresh tokens | Rotated on every use, stored hashed, revocable |
| Token revocation | Revoke list for admin accounts and stolen tokens |
| Passwords | Strong hashing (bcrypt or Argon2), never stored in plain text, lockout after repeated failures |

Never store JWTs, passwords or card data in logs.

## 3. Authorization (what may you do?)
Two checks at two layers:

| Layer | Check | Example |
|---|---|---|
| API Gateway | Role from JWT claims | Only `admin` can call stock adjustment endpoints |
| Each service | Object ownership | A customer can only read their own reservation, payment and order |

Roles (assumption): `customer`, `support`, `admin`, `system`.

| Role | Allowed |
|---|---|
| customer | Browse, reserve, pay, view own orders |
| support | Read orders and payments, trigger refunds with approval |
| admin | Manage sale and stock configuration, view audit logs |
| system | Internal service accounts (reconciler, sweeper) |

Rule: least privilege. Every role gets only what it needs.

## 4. Service-to-service authentication
- mTLS between all internal services, with short-lived certificates issued automatically (service mesh or sidecar).
- Service identity is checked on every call.
- Allow-list per service. Example: only Payment Service may call the payment gateway client, only the order consumer may write orders.
- Each service has its own database user with only the permissions it needs.

## 5. Object-level checks (stopping IDOR)
Never trust an ID in the URL alone. The service compares `customer_id` from the JWT with the owner of the requested resource. A mismatch returns 403 (or 404 to avoid leaking existence).

## 6. Error responses
| Code | Meaning |
|---|---|
| 401 Unauthorized | Missing, invalid or expired token |
| 403 Forbidden | Valid token, not allowed |
| 429 Too Many Requests | Rate limit hit |

## 7. Webhook authentication
Payment gateway callbacks are verified using the gateway's signature, then the status is confirmed by querying the gateway before marking a payment SUCCESS.

## 8. Trade-offs
- Short JWTs and mTLS add operational work (token refresh, certificate rotation) and a small latency cost. We accept this for stronger security and automate it with a service mesh.
- Stateless JWT verification scales well but makes instant revocation harder, so we keep a revoke list for sensitive accounts.

