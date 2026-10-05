# Secrets Management

## 1. What counts as a secret
Database credentials, Redis password, broker credentials, payment gateway API key, webhook signing secret, JWT signing keys, TLS private keys, encryption keys.

## 2. Rules
1. No secrets in source code, container images, config files in Git, or logs.
2. All secrets live in a central secrets vault.
3. Each service can read only its own secrets (least privilege).
4. Secrets are rotated automatically (30 days assumption, shorter for database credentials where supported).
5. Access to the vault is audited.

## 3. How services get secrets
| Step | Behaviour |
|---|---|
| Startup | Service authenticates to the vault using its service identity and fetches its secrets |
| Runtime | Secrets cached in memory only |
| Rotation | Service refreshes secrets without restart where possible |
| Shutdown | Memory cleared with the process |

## 4. Separation by environment
Development, test and production use different secrets and different vault paths. Production secrets are never used in tests.

## 5. Key management
- JWT signing keys: rotated with key IDs (kid) so old tokens validate during rotation.
- Encryption keys managed by a key management service, not stored with the data they protect.

## 6. If a secret leaks
1. Revoke and rotate immediately.
2. Search the audit log for use of the leaked secret.
3. Review access, then add a detection rule.

## 7. Prevention
- Secret scanning in the repository and in CI.
- Pre-commit hooks.
- Never print environment variables in logs.

## 8. Failure behaviour
| Failure | Behaviour |
|---|---|
| Vault down | Running instances keep working with cached secrets. New instances cannot start until the vault returns. The vault runs highly available. |

## 9. Trade-offs
The vault is a critical dependency, so it needs high availability and good access control. In exchange, we remove secrets from code and gain rotation and auditing.
