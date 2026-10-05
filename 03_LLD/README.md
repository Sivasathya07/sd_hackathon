# SALESTORM — Low Level Design

## 1. LLD Overview

This document describes the low-level design of the SALESTORM flash-sale system.

The design focuses on:

* Inventory reservation and concurrency control
* Payment processing
* Order creation
* Failure recovery
* Event-driven communication
* Idempotency
* Resilience and fault tolerance

The major components are represented using class, sequence, and state diagrams.

---

## 2. Module / Class Diagram

The class diagram represents the core domain entities, repositories, and external payment abstraction.

### Main Entities

* **Inventory** — Maintains available, reserved, and sold quantities.
* **InventoryReservation** — Represents a temporary stock reservation.
* **Order** — Represents the customer's confirmed purchase.
* **PaymentTransaction** — Maintains payment transaction details.
* **PaymentGateway** — Abstraction for external payment providers.
* **Repositories** — Provide persistence access for inventory and orders.

### Diagram

The source diagram is available at:

`diagrams/01_class_diagram.puml`

### Important Design Decisions

* Inventory uses a `version` field for optimistic locking.
* Reservations have an expiry time to prevent stock from being held indefinitely.
* Payment transactions contain an idempotency key to avoid duplicate charges.
* Payment providers are accessed through the `PaymentGateway` abstraction.

---

## 3. State Machine

The reservation lifecycle is represented using the following states:

```text
AVAILABLE
    ↓
RESERVED
    ↓
PAYMENT_PENDING
    ↓
CONFIRMED
    ↓
SOLD
```

Failure paths:

```text
PAYMENT_PENDING → RELEASED → AVAILABLE

RESERVED → EXPIRED → AVAILABLE
```

### State Descriptions

| State           | Description                                         |
| --------------- | --------------------------------------------------- |
| AVAILABLE       | Stock is available for reservation                  |
| RESERVED        | Stock has been temporarily reserved                 |
| PAYMENT_PENDING | Payment processing is in progress                   |
| CONFIRMED       | Payment was successful and reservation is confirmed |
| RELEASED        | Reserved stock was released after payment failure   |
| EXPIRED         | Reservation exceeded its TTL                        |
| SOLD            | Order has been fulfilled                            |

### Diagram

The source diagram is available at:

`diagrams/05_state_diagram.puml`

---

# 4. Sequence Diagrams

## 4.1 Purchase & Inventory Reservation

The reservation flow prevents overselling during high-concurrency flash sales.

### Flow

1. Customer sends a checkout reservation request.
2. API Gateway forwards the request to the Checkout Facade.
3. Inventory Service checks the idempotency key.
4. Inventory quantity and version are read from the database.
5. The service performs an optimistic-lock update.
6. If the version matches, the stock is reserved.
7. A reservation record is created with a 15-minute expiry.
8. If stock is unavailable, the request returns a conflict response.
9. If a concurrent update causes a version mismatch, the operation is retried.

### Key Mechanisms

* Idempotency
* Optimistic locking
* Reservation TTL
* Retry mechanism

Source:

`diagrams/02_reservation_sequence.puml`

---

## 4.2 Payment Processing

The payment flow uses an idempotency check and circuit breaker to protect the system from duplicate requests and payment-provider failures.

### Flow

1. Customer sends a payment request.
2. Payment Service checks the idempotency key.
3. Previously processed transactions return the existing result.
4. New transactions are passed through the Circuit Breaker.
5. Payment Gateway Adapter communicates with the external payment provider.
6. Successful payments publish `PaymentSucceededEvent`.
7. Failed payments publish `PaymentFailedEvent`.
8. Timeout cases publish `PaymentPendingReconciliationEvent`.
9. If the circuit is open, the request fails fast instead of repeatedly calling the payment provider.

### Key Mechanisms

* Idempotency
* Adapter pattern
* Circuit breaker
* Event-driven communication
* Timeout handling

Source:

`diagrams/03_payment_sequence.puml`

---

## 4.3 Order Creation & Recovery

Order creation is event-driven and designed to recover from temporary database failures.

### Flow

1. Payment Service publishes `PaymentSucceededEvent`.
2. Order Service consumes the event.
3. Order Service checks whether an order already exists for the reservation.
4. If the order exists, the event is acknowledged.
5. If the order does not exist, a new confirmed order is created.
6. If database creation fails, the event is retried.
7. After repeated failures, the event is moved to the Dead Letter Queue.
8. A reconciliation job processes the failed event.
9. Payment failures trigger a compensating inventory action.
10. Reserved stock is released back into available stock.

### Key Mechanisms

* Kafka event processing
* Retry mechanism
* Dead Letter Queue
* Reconciliation
* Idempotent order creation
* Compensating transaction

Source:

`diagrams/04_order_recovery_sequence.puml`

---

# 5. SOLID Principles Mapping

The design follows SOLID principles to keep the system modular and maintainable.

## 5.1 Single Responsibility Principle

Each component has a focused responsibility.

| Component             | Responsibility                          |
| --------------------- | --------------------------------------- |
| PaymentProcessor      | Payment execution                       |
| PaymentGatewayAdapter | Communication with payment provider     |
| PaymentRepository     | Payment persistence                     |
| InventoryService      | Inventory reservation logic             |
| OrderService          | Order creation and lifecycle management |

This prevents business logic, external communication, and persistence from becoming tightly coupled.

---

## 5.2 Open/Closed Principle

The payment system is designed around the `PaymentGateway` abstraction.

New payment providers can be added without modifying the Checkout Facade.

Example:

```text
PaymentGateway
      |
      +--- StripeAdapter
      |
      +--- RazorpayAdapter
      |
      +--- ApplePayAdapter
```

---

## 5.3 Liskov Substitution Principle

Different payment gateway implementations can be substituted wherever the `PaymentGateway` interface is expected.

For example:

```text
PaymentGateway
     ↓
StripeAdapter
     ↓
RazorpayAdapter
```

The calling service does not need to know which provider implementation is being used.

---

## 5.4 Interface Segregation Principle

Large interfaces can be separated into smaller role-specific interfaces.

For example:

```text
InventoryReadRepository
InventoryWriteRepository
```

A component that only reads inventory does not need to depend on write operations.

---

## 5.5 Dependency Inversion Principle

High-level business components depend on abstractions rather than concrete infrastructure implementations.

For example:

```text
CheckoutFacade
      ↓
InventoryService
      ↓
InventoryRepository
```

The business logic does not directly depend on a specific database implementation.

---

# 6. Design Patterns

## 6.1 Strategy Pattern

The Strategy pattern can be used when different payment or pricing strategies need to be selected dynamically.

Example:

```text
PaymentStrategy
      |
      +--- CardPaymentStrategy
      +--- UpiPaymentStrategy
      +--- WalletPaymentStrategy
```

This allows different payment strategies to be changed without modifying the main checkout logic.

---

## 6.2 Factory Pattern

A Factory can create the appropriate payment gateway implementation.

Example:

```text
PaymentGatewayFactory
        |
        +--- StripeAdapter
        +--- RazorpayAdapter
```

The calling service does not need to directly instantiate provider-specific implementations.

---

## 6.3 State Pattern

The reservation lifecycle is represented through well-defined states.

```text
AVAILABLE
    ↓
RESERVED
    ↓
PAYMENT_PENDING
    ↓
CONFIRMED
    ↓
SOLD
```

Failure states such as `RELEASED` and `EXPIRED` return the inventory to the available state.

---

## 6.4 Observer Pattern

The event-driven architecture follows the Observer-style communication model.

Example:

```text
Payment Service
      |
      | PaymentSucceededEvent
      ↓
Kafka
      |
      +----> Order Service
      |
      +----> Notification Service
```

Producers do not need direct knowledge of every consumer.

---

## 6.5 Adapter Pattern

The Payment Gateway Adapter isolates provider-specific APIs from the internal payment service.

```text
Payment Service
       ↓
PaymentGateway
       ↓
PaymentGatewayAdapter
       ↓
External Payment Provider
```

This prevents external provider-specific logic from leaking into the core business logic.

---

## 6.6 Repository Pattern

Repositories provide an abstraction over database operations.

Examples:

```text
InventoryRepository
OrderRepository
PaymentRepository
```

Business services interact with repositories instead of directly executing database operations.

---

## 6.7 Facade Pattern

The Checkout Facade provides a simplified entry point for the checkout workflow.

```text
Customer
   ↓
Checkout Facade
   ↓
+--------------------+
| Inventory Service  |
| Payment Service    |
| Order Service      |
+--------------------+
```

The client does not need to coordinate every internal service directly.

---

## 6.8 Circuit Breaker Pattern

The payment service uses a circuit breaker when communicating with an external payment provider.

```text
CLOSED
   ↓
Repeated Failures
   ↓
OPEN
   ↓
Fail Fast
   ↓
HALF-OPEN
   ↓
Recovery Test
   ↓
CLOSED
```

This prevents repeated requests to an unhealthy external payment provider.

---

# 7. Key LLD Decisions

## 7.1 Optimistic Locking

Inventory uses a version number to handle concurrent reservation requests.

Conceptually:

```text
Read version = 10

UPDATE inventory
SET available_quantity = available_quantity - 1,
    version = version + 1
WHERE product_id = ?
AND version = 10
```

If another transaction has already changed the version, the update fails and the reservation can be retried.

This helps prevent overselling during high-concurrency flash sales.

---

## 7.2 Idempotency

Important operations use an idempotency key.

Examples:

```text
X-Idempotency-Key
```

If the same request is received multiple times, the system can return the previous result instead of performing the operation again.

This is particularly important for:

* Inventory reservation
* Payment processing
* Order creation

---

## 7.3 Reservation TTL

A reservation is temporary.

Example:

```text
Reservation Created
       ↓
15 Minute TTL
       ↓
Payment Completed → CONFIRMED

OR

TTL Expired → RELEASED
```

This prevents abandoned checkouts from permanently locking inventory.

---

## 7.4 Saga / Compensating Transaction

Distributed operations may partially succeed.

For example:

```text
Reserve Inventory
       ↓
Payment
       ↓
Payment Failed
       ↓
Release Inventory
```

Instead of requiring a distributed database transaction, the system performs a compensating action to restore the previous business state.

---

## 7.5 Dead Letter Queue

Events that repeatedly fail processing are moved to a Dead Letter Queue.

```text
Kafka Event
     ↓
Order Service
     ↓
Failure
     ↓
Retry
     ↓
Retry
     ↓
Retry
     ↓
DLQ
     ↓
Reconciliation Job
```

This prevents permanently failing events from blocking normal event processing.

---

## 7.6 Circuit Breaker

The Circuit Breaker protects the application from an unhealthy external payment provider.

Instead of continuously sending requests during an outage, the circuit opens and requests fail fast.

This reduces cascading failures and allows the external provider time to recover.

---

# 8. LLD Summary

The SALESTORM design combines object-oriented design with event-driven architecture.

The major design goals are:

* Prevent inventory overselling
* Avoid duplicate payments and reservations
* Handle concurrent requests safely
* Isolate external payment providers
* Recover from temporary failures
* Support asynchronous event processing
* Release inventory when transactions fail
* Keep business logic independent from infrastructure

The design uses SOLID principles and established design patterns to keep the system modular, extensible, and resilient under flash-sale traffic.
