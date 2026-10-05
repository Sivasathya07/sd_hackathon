# Input Validation

Owner: [Your Name] (Student 4)

## 1. Principle
Never trust input. Validate at the edge, then validate again inside each service.

## 2. Layers
| Layer | Validation |
|---|---|
| WAF | Block known attack patterns (SQL injection, XSS signatures), oversized requests |
| API Gateway | Schema validation from the OpenAPI spec: required fields, types, lengths, formats, reject unknown fields |
| Service | Business rules: quantity limits, sale active, product exists, ownership |
| Database | Constraints: NOT NULL, CHECK, UNIQUE, foreign keys |

## 3. Rules
| Input | Rule |
|---|---|
| Idempotency-Key | Required on POST, UUID format, max 64 characters |
| product_id, sale_id | Positive integers or UUIDs, must exist |
| quantity | Integer, at least 1, at most the per-customer limit (for a flash sale, 1) |
| customer_id | Taken from the JWT, never from the request body |
| payment token | Matches the gateway token format, never raw card data |
| Strings | Length limits, allowed character sets, trimmed |
| Request size | Maximum body size enforced at the gateway |

## 4. Injection protection
- Parameterised queries and prepared statements only. No string-built SQL.
- Output encoding for any user text shown in pages and emails.
- No dynamic code execution or unsafe deserialization of untrusted input.
- Redis Lua scripts receive values as arguments, never built from user strings.

## 5. Safe failure
| Case | Response |
|---|---|
| Malformed or invalid body | 400 Bad Request with a clear error code, no internal details |
| Business rule violation | 409 or 422 with a stable error code |
| Unauthorised | 401 or 403 |

Error messages never reveal stack traces, SQL or internal hostnames.

## 6. Messages and events
Consumers validate message schema and version. Invalid messages are non-retryable and go straight to the DLQ.

## 7. Trade-offs
Double validation (gateway and service) costs a little CPU but stops bypass if one layer is misconfigured.

## 8. Jury answer
Validation happens at the edge for speed, and again in services and the database for correctness. The customer identity always comes from the token, never from the body.
