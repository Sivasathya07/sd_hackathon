# SALESTORM – HLD Diagrams

This folder contains the five High-Level Design diagrams for the SALESTORM flash-sale system.

## Diagrams

### 1. System Context Diagram
`01_System_Context_Diagram.png`

Shows the interaction between the SALESTORM system and external actors such as customers, payment gateway, shipment provider, and notification provider.

### 2. HLD Architecture
`02_HLD_Architecture.png`

Shows the overall SALESTORM architecture, including the API Gateway, core services, Redis, database, message broker, external systems, and observability.

### 3. Container Diagram
`03_Container_Diagram.png`

Shows the major application services and infrastructure containers and how they communicate during the purchase flow.

### 4. Component Diagram
`04_Component_Diagram.png`

Shows the internal components involved in checkout, inventory, reservation, payment, and order processing.

### 5. Deployment Diagram
`05_Deployment_Diagram.png`

Shows the deployment of SALESTORM across the cloud environment, including load balancing, multiple service instances, Redis, database, message broker, monitoring, logging, and tracing.

## Files

```text
02_HLD/
├── 01_System_Context_Diagram.png
├── 02_HLD_Architecture.png
├── 03_Container_Diagram.png
├── 04_Component_Diagram.png
└── 05_Deployment_Diagram.png
