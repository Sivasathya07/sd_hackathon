# SALESTORM – Assumptions

## 1. Traffic Assumptions

- The flash sale may receive 10,000 concurrent purchase requests.
- The architecture should be capable of reasoning about traffic up to 500,000 requests/sec.
- Traffic can increase suddenly during the flash sale.

## 2. Inventory Assumptions

- The critical product has 100 available units.
- Inventory must never become negative.
- Inventory reservation is temporary.
- Expired or failed reservations are released.

## 3. Payment Assumptions

- Payment processing is handled through an external payment gateway.
- Payment requests may succeed, fail or timeout.
- Duplicate payment requests may occur.
- Payment operations require idempotency.

## 4. Service Assumptions

- Application services can be horizontally scaled.
- Some downstream services may temporarily become unavailable.
- Asynchronous messaging can be used for reliable processing and recovery.

## 5. Data Assumptions

- Inventory, reservation, payment and order data require strong consistency at their critical transaction boundaries.
- Each reservation has a unique reservation identifier.
- Critical repeated requests use an idempotency key.

## 6. Reliability Assumptions

- Failed asynchronous messages may require retry processing.
- Permanently failed messages may be handled through a dead-letter mechanism.
- Distributed workflow failures may require reconciliation or compensation.

## 7. Security Assumptions

- Customers must be authenticated for protected operations.
- Communication between clients and services uses secure HTTPS communication.
- Payment information is handled securely.

## 8. Design Priority

For the flash-sale purchase flow, preventing inventory overselling and maintaining transaction correctness take priority over allowing an incorrect purchase.
