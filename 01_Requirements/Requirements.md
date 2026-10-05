# SALESTORM – Requirements

## 1. Problem Statement

SALESTORM is a high-scale e-commerce flash-sale platform where 10,000 customers may simultaneously attempt to purchase a product with only 100 units available.

The system must handle the traffic surge while preventing inventory overselling, duplicate reservations and duplicate orders. It must also process payments safely and maintain a reliable order lifecycle even when services fail.

## 2. Business Objective

The primary objective is to design a scalable, reliable and fault-tolerant e-commerce system capable of handling flash-sale traffic while maintaining inventory consistency and reliable payment and order processing.

## 3. Functional Requirements

### FR1 – Product Discovery
Customers should be able to browse products and view product information.

### FR2 – Cart Management
Customers should be able to add products to their cart and manage cart items.

### FR3 – Inventory Check
The system should check product availability before allowing a purchase.

### FR4 – Inventory Reservation
The system should temporarily reserve available inventory for a customer.

### FR5 – Reservation Expiry
Expired or unpaid reservations should be released back to available inventory.

### FR6 – Checkout
The system should initiate checkout for a valid reservation.

### FR7 – Payment Processing
The system should process customer payments securely.

### FR8 – Payment Failure Handling
The system should handle payment failures and release the associated reservation.

### FR9 – Payment Timeout Handling
The system should safely handle payment timeouts without creating duplicate payments.

### FR10 – Order Creation
A successful payment should eventually result in a valid order.

### FR11 – Duplicate Request Prevention
Repeated purchase or payment requests should not create duplicate reservations, payments or orders.

### FR12 – Order Lifecycle
The system should maintain the order through its lifecycle from creation to delivery.

### FR13 – Shipment
The system should support shipment processing for confirmed orders.

### FR14 – Notification
The system should send relevant notifications to customers.

### FR15 – Delivery Tracking
The system should support delivery tracking.

## 4. Non-Functional Requirements

### NFR1 – Scalability
The architecture should support approximately 10,000 requests/sec under normal traffic and reason about traffic up to 500,000 requests/sec during extreme flash-sale conditions.

### NFR2 – Inventory Consistency
The system must never oversell available inventory. For 100 available units, no more than 100 units can be successfully sold.

### NFR3 – Reliability
The system should provide defined recovery mechanisms for service, database and payment failures.

### NFR4 – Availability
The system should continue operating when non-critical downstream services temporarily fail.

### NFR5 – Performance
The system should efficiently handle high-volume concurrent requests with controlled latency.

### NFR6 – Security
The system should provide authentication, authorization, HTTPS communication, rate limiting, input validation and secure handling of payment information.

### NFR7 – Observability
The system should provide monitoring, structured logging and distributed tracing for critical transactions.

### NFR8 – Fault Tolerance
The architecture should support timeout handling, retries, circuit breakers, asynchronous processing and recovery mechanisms.

## 5. Success Criteria

The system should satisfy the following:

- 10,000 concurrent purchase requests can be handled through controlled scaling.
- 100 available units must never result in more than 100 successful sales.
- Unpaid or expired reservations must be released.
- Duplicate payment requests must not create duplicate transactions.
- Successful purchases should eventually reach a valid order state.
- Defined recovery paths should exist for service and payment failures.
- The architecture should support horizontal scaling during peak traffic.
