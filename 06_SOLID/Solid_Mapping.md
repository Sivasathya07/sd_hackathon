# SALESTORM - SOLID Principles Mapping

## Overview

The SALESTORM system follows SOLID principles to keep the architecture modular, maintainable, testable, and easy to extend.

The following mappings describe how SOLID principles are applied in the proposed system design.

---

## 1. Single Responsibility Principle (SRP)

> A class should have one primary responsibility.

| Component               | Responsibility                                     |
| ----------------------- | -------------------------------------------------- |
| `PaymentProcessor`      | Handles payment processing logic                   |
| `PaymentGatewayAdapter` | Communicates with the external payment provider    |
| `PaymentRepository`     | Handles payment persistence                        |
| `InventoryService`      | Handles inventory reservation and release          |
| `OrderService`          | Handles order creation and order status management |

### Example

The `PaymentGatewayAdapter` is responsible only for communication with the external payment provider. Database persistence is handled separately by the repository layer.

This prevents one class from becoming responsible for payment processing, external API communication, and database operations simultaneously.

---

## 2. Open/Closed Principle (OCP)

> Software entities should be open for extension but closed for modification.

The system uses the `PaymentGateway` interface to support multiple payment providers.

```text
PaymentGateway
      |
      +---- StripeAdapter
      |
      +---- RazorpayAdapter
      |
      +---- FuturePaymentAdapter
```

A new payment provider can be added by creating another implementation of `PaymentGateway`.

The existing checkout and payment-processing logic does not need to be modified.

### Benefit

New payment providers can be introduced without changing existing business logic.

---

## 3. Liskov Substitution Principle (LSP)

> Subtypes should be replaceable for their base types without breaking the system.

Different payment gateway implementations follow the same `PaymentGateway` contract.

```text
PaymentGateway
      |
      +---- StripeAdapter
      |
      +---- RazorpayAdapter
```

The payment service can work with either implementation through the common interface.

For example:

```text
PaymentGateway gateway = StripeAdapter
```

can be replaced by:

```text
PaymentGateway gateway = RazorpayAdapter
```

without changing the payment-processing flow.

### Benefit

Implementations remain interchangeable and follow a consistent contract.

---

## 4. Interface Segregation Principle (ISP)

> Clients should not be forced to depend on interfaces they do not use.

Repository responsibilities can be separated into smaller interfaces instead of creating one large repository interface.

For example:

```text
InventoryReadRepository
        |
        +-- findByProductId()

InventoryWriteRepository
        |
        +-- reserve()
        +-- release()
        +-- save()
```

A component that only needs to read inventory does not need to depend on write operations.

### Benefit

Smaller interfaces reduce unnecessary dependencies and make components easier to test.

---

## 5. Dependency Inversion Principle (DIP)

> High-level modules should depend on abstractions rather than concrete implementations.

The system uses interfaces for important infrastructure dependencies.

Example:

```text
CheckoutFacade
      |
      v
InventoryService
      |
      v
InventoryRepository
      |
      +---- PostgreSQL implementation
```

Similarly:

```text
PaymentService
      |
      v
PaymentGateway
      |
      +---- StripeAdapter
      +---- RazorpayAdapter
```

The business logic does not directly depend on a specific database or payment provider.

### Benefit

Infrastructure implementations can be replaced without changing the core business logic.

---

# SOLID Summary

| Principle | SALESTORM Application                                                          |
| --------- | ------------------------------------------------------------------------------ |
| **SRP**   | Separates payment, inventory, order, adapter, and persistence responsibilities |
| **OCP**   | New payment providers can be added through `PaymentGateway`                    |
| **LSP**   | Different payment gateway implementations are interchangeable                  |
| **ISP**   | Repository interfaces can be divided according to client requirements          |
| **DIP**   | Business services depend on interfaces instead of concrete infrastructure      |

---

# Overall Benefit

Applying SOLID principles helps SALESTORM achieve:

* Better separation of responsibilities
* Easier testing
* Lower coupling
* Easier addition of new payment providers
* Easier replacement of infrastructure components
* Better maintainability
* Better scalability

> **Note:** These mappings represent the intended/proposed architecture of SALESTORM. They should not be interpreted as proof that every abstraction is already implemented in production code.
