# SALESTORM - Design Patterns Mapping

## Overview

The SALESTORM architecture uses established design patterns to improve flexibility, maintainability, extensibility, and resilience.

The following patterns are mapped to the proposed system design.

---

## 1. Strategy Pattern

### Purpose

The Strategy Pattern allows different algorithms or behaviors to be selected without changing the code that uses them.

### SALESTORM Application

The payment system can support multiple payment providers through a common `PaymentGateway` abstraction.

```text
PaymentService
      |
      v
PaymentGateway
      |
      +---- StripeAdapter
      |
      +---- RazorpayAdapter
      |
      +---- FuturePaymentAdapter
```

Each payment provider can implement the same payment contract.

### Benefit

* Easy to add new payment providers
* Reduces conditional logic
* Keeps payment processing flexible

---

## 2. Factory Pattern

### Purpose

The Factory Pattern centralizes object creation and hides the implementation details from the client.

### SALESTORM Application

A payment gateway factory can select the required payment provider based on configuration or request information.

```text
PaymentGatewayFactory
        |
        +---- StripeAdapter
        |
        +---- RazorpayAdapter
```

Example:

```text
PaymentGateway gateway =
    PaymentGatewayFactory.create(provider);
```

### Benefit

* Centralizes object creation
* Reduces dependency on concrete classes
* Makes provider selection easier to maintain

---

## 3. State Pattern

### Purpose

The State Pattern represents different states of an object and controls how its behavior changes based on its current state.

### SALESTORM Application

Inventory reservations move through defined lifecycle states:

```text
AVAILABLE
    |
    v
RESERVED
    |
    v
PAYMENT_PENDING
    |
    +----> CONFIRMED
    |
    +----> RELEASED
    |
    +----> EXPIRED
```

The reservation state determines which transitions are allowed.

### Benefit

* Makes lifecycle transitions explicit
* Prevents invalid state changes
* Simplifies reservation lifecycle management

---

## 4. Observer Pattern

### Purpose

The Observer Pattern allows one component to publish an event while multiple interested components react to that event.

### SALESTORM Application

SALESTORM uses event-driven communication through Kafka.

For example:

```text
Payment Service
      |
      | PaymentSucceededEvent
      v
Kafka
      |
      +----> Order Service
      |
      +----> Notification Service
      |
      +----> Analytics Service
```

The payment service does not need to directly call every consumer.

### Benefit

* Loose coupling between services
* Supports asynchronous processing
* New event consumers can be added easily

---

## 5. Adapter Pattern

### Purpose

The Adapter Pattern allows incompatible interfaces to work together.

### SALESTORM Application

External payment providers may expose different APIs.

The `PaymentGatewayAdapter` converts the internal payment request into the format expected by the external provider.

```text
Payment Service
      |
      v
PaymentGatewayAdapter
      |
      v
External Payment Provider
```

Different provider-specific adapters can hide external API differences from the payment service.

### Benefit

* Isolates external API differences
* Prevents vendor-specific logic from spreading through the system
* Makes provider replacement easier

---

## 6. Repository Pattern

### Purpose

The Repository Pattern separates business logic from data-access logic.

### SALESTORM Application

Repositories provide an abstraction over database operations.

```text
OrderService
      |
      v
OrderRepository
      |
      v
Order Database
```

Similarly:

```text
InventoryService
      |
      v
InventoryRepository
      |
      v
Inventory Database
```

### Benefit

* Separates persistence from business logic
* Improves testability
* Makes database implementation easier to change

---

## 7. Facade Pattern

### Purpose

The Facade Pattern provides a simplified interface to a complex subsystem.

### SALESTORM Application

The `CheckoutFacade` provides a simplified entry point for the checkout process.

```text
Customer
   |
   v
CheckoutFacade
   |
   +----> InventoryService
   |
   +----> PaymentService
   |
   +----> OrderService
```

The client does not need to directly coordinate every underlying service.

### Benefit

* Simplifies client interaction
* Hides subsystem complexity
* Provides a clean API boundary

---

## 8. Circuit Breaker Pattern

### Purpose

The Circuit Breaker Pattern prevents repeated calls to an unavailable external service.

### SALESTORM Application

The payment service uses a circuit breaker when communicating with the external payment provider.

```text
Payment Service
      |
      v
Circuit Breaker
      |
      v
External PSP
```

The circuit can have states such as:

```text
CLOSED
   |
   | Repeated failures
   v
OPEN
   |
   | Recovery timeout
   v
HALF_OPEN
   |
   | Successful request
   v
CLOSED
```

When the circuit is open, requests fail fast instead of continuously calling the unavailable provider.

### Benefit

* Prevents cascading failures
* Reduces unnecessary external calls
* Improves system resilience
* Allows automatic recovery

---

# Design Pattern Summary

| Pattern             | SALESTORM Component        | Main Purpose                        |
| ------------------- | -------------------------- | ----------------------------------- |
| **Strategy**        | Payment Gateway            | Select payment provider behavior    |
| **Factory**         | Payment Gateway Factory    | Centralize provider object creation |
| **State**           | Inventory Reservation      | Manage reservation lifecycle        |
| **Observer**        | Kafka Events               | Event-driven communication          |
| **Adapter**         | Payment Gateway Adapter    | Integrate external payment APIs     |
| **Repository**      | Inventory/Order Repository | Abstract data persistence           |
| **Facade**          | Checkout Facade            | Simplify checkout operations        |
| **Circuit Breaker** | Payment Service            | Protect against PSP failures        |

---

# Overall Benefits

These patterns help SALESTORM achieve:

* Loose coupling between components
* Better separation of concerns
* Easier extension of payment providers
* Improved testability
* Cleaner service boundaries
* Better fault tolerance
* Easier maintenance
* Scalable event-driven communication

> **Note:** These are design-pattern mappings for the proposed SALESTORM architecture. They describe how the patterns are intended to be used and should not be interpreted as proof that every pattern is fully implemented in the current codebase.
