# ADR-003: Modular Services Architecture

- **Status:** Accepted
- **Date:** 2026-11-18

## 1. Context

NetGuard contains several responsibilities, including telemetry collection, processing, ML inference, incident management and API access.

These responsibilities have different resource requirements and deployment needs.

A single application could simplify development and deployment, but it would limit independent scaling and deployment.

On the other hand, creating a separate microservice for every small function would introduce unnecessary distributed-system complexity.

## 2. Alternatives Considered

### Alternative A: Modular monolith

Organize the application into well-defined modules but deploy them as a single application.

**Advantages:**
- Simpler deployment and debugging.
- Easier internal communication.
- Lower operational complexity.

**Disadvantages:**
- Components cannot be scaled independently.
- Changes to one module may require redeploying the entire application.
- Different workloads share the same deployment boundary.

### Alternative B: Full microservices architecture

Deploy every major responsibility, potentially including small individual functions, as independent services.

**Advantages:**
- Independent deployment and scaling.
- Clear responsibility boundaries.
- Technology flexibility.

**Disadvantages:**
- Increased infrastructure and monitoring complexity.
- More network communication and failure points.
- Greater development and maintenance effort.
- Risk of creating a distributed monolith through excessive coupling.

### Alternative C: Hybrid modular services

Create a limited number of independently deployable services, each containing cohesive internal modules.

**Advantages:**
- Independent scaling and deployment for important workloads.
- Clear responsibility boundaries.
- Avoids unnecessary service fragmentation.
- Balances modularity with operational simplicity.

**Disadvantages:**
- Requires careful service-boundary design.
- Still introduces distributed-system concerns such as network failures and message delivery.
- Requires coordination between service contracts.

## 3. Decision

We adopt a hybrid modular-services architecture.

Major responsibilities will be deployed independently when they have meaningful differences in scaling, deployment or operational requirements.

Small, closely related functions will remain grouped within their responsible service.

Kafka will provide asynchronous communication between relevant services.

## 4. Consequences

NetGuard can scale and deploy important workloads independently while avoiding excessive microservice fragmentation.

The architecture will require explicit service contracts, observability and failure-handling mechanisms.

Service boundaries may be reconsidered as the platform evolves.