# Payment Data Security

## 1. Principle
The safest card data is data we never receive.

## 2. Design
| Item | Design |
|---|---|
| Card entry | Hosted card fields or redirect by the payment gateway. Card number, CVV and expiry never touch our servers |
| What we receive | A payment token from the gateway |
| What we store | Token, last 4 digits, card brand, transaction_ref, amount, status |
| What we never store | Full card number, CVV, PIN |
| PCI scope | Minimised, because we do not process or store card data |
| Transport | HTTPS only, API key from the secrets vault |

## 3. Safe payment handling
| Control | Purpose |
|---|---|
| Unique transaction_ref per payment | Reconcile with the gateway, prevent duplicate charges |
| Idempotency-Key on payment requests | Duplicate clicks return the same result |
| Payment row written PENDING before calling the gateway | A record exists even if we crash mid-call |
| Amount computed server-side | Client cannot change the price |
| Webhook signature verification | Stops fake payment callbacks |
| Status confirmation by querying the gateway | Never trust a callback alone |
| Refund_ref derived from transaction_ref | Refunds are idempotent |

## 4. Fake payment callback attack
Attacker calls our webhook claiming "payment success".
Defence: verify the signature, then query the gateway for the true status before marking SUCCESS.

## 5. Logging
Never log card numbers, CVV, full tokens or secrets. Mask tokens (show last 4). The log collector masks sensitive fields as a second layer.

## 6. Access
- Only Payment Service can use the gateway API key.
- Support staff see masked data only.
- Refunds need role `support` or `admin` and are audited.

## 7. Data retention
Keep payment records as required for accounting and disputes. Delete or anonymise personal data per policy.

## 8. Trade-offs
Using hosted fields reduces control over the card form's look, but greatly reduces risk and compliance work.

