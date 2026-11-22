# ADR-008: Telemetry Collector Architecture

**Status:** Accepted

**Date:** 2026-11-21

## Context

NetGuard receives telemetry from multiple sources, including network devices, monitoring agents, historical data replay and synthetic generators.

These sources may use different communication protocols and payload formats. The platform therefore needs a collection layer that provides a unified ingestion interface to downstream components.

Two collection mechanisms are required:

- Pull-based collection, where NetGuard periodically requests measurements.
- Push-based collection, where external sources send measurements to NetGuard.

The architecture must support independent scaling, failure isolation, shared normalization logic and reliable publication to Kafka.

## Decision

We adopt a hybrid collector architecture composed of two independently deployable services:

1. Pull Collector.
2. Push Collector.

Both services reuse shared components for event normalization, validation, canonical schema management and Kafka publishing.

### 1. Collector Deployment

The Pull Collector and Push Collector are deployed independently.

The Pull Collector supports multiple replicas with coordinated device assignments to avoid duplicate polling.

The Push Collector supports multiple replicas behind a Kubernetes Service, which distributes incoming requests among available instances.

### 2. Canonical Event Schema

Both collectors transform source-specific measurements into a common canonical telemetry event format.

Downstream services must not depend on the original source payload format.

Each event preserves its event identifier, source metadata, link identifier and original measurement timestamp.

### 3. Kafka Integration

Both collectors publish normalized events to one initial Kafka topic:

`telemetry.raw`

The initial design does not introduce separate topics for pull and push traffic.

Source metadata allows downstream components and monitoring systems to distinguish the ingestion origin.

### 4. Buffering and Reliability

The initial implementation uses bounded in-memory queues.

Collectors consider an event successfully published after receiving the configured Kafka broker acknowledgement.

When publishing cannot keep up with incoming traffic, the collectors must apply explicit backpressure rather than silently discarding events.

A durable disk-backed spool may be introduced in a future iteration.

### 5. Duplicate Handling

Events carry a unique `event_id` that is propagated through the processing pipeline.

Consumers and persistence boundaries must handle duplicate delivery safely where repeated processing could create incorrect side effects.

Kafka producer idempotence and application-level idempotency are complementary mechanisms, not substitutes for one another.

## Alternatives Considered

### A. One unified collector service

A single service handles both pull and push ingestion.

**Advantages:**
- Simpler deployment.
- Shared runtime and infrastructure.
- Less operational overhead.

**Disadvantages:**
- Pull and push workloads share the same process and resource limits.
- A process-level failure affects both collection mechanisms.
- Independent scaling and release management are less flexible.

### B. Completely separate collector implementations

Pull and push collection are implemented as independent services with duplicated logic.

**Advantages:**
- Strong deployment independence.
- Independent scaling and configuration.

**Disadvantages:**
- Duplicated normalization and validation logic.
- Greater maintenance burden.
- Risk of inconsistent event formats and behavior.

### C. Independent collector services with shared components

Pull and push are separate deployable services but reuse common libraries and the canonical event schema.

**Advantages:**
- Independent deployment and scaling.
- Better failure isolation.
- Shared normalization and publishing logic.
- Consistent event contracts.

**Disadvantages:**
- Requires shared component maintenance.
- Introduces additional deployment and operational complexity.
- Pull replicas require device assignment coordination.

**Selected alternative: C.**

## Consequences

### Positive

- Pull and push workloads can scale independently.
- Failure of one collector service does not inherently stop the other.
- Downstream components receive a consistent event format.
- Both ingestion paths share one internal Kafka topic.
- Source provenance is preserved for debugging and analytics.
- Bounded queues decouple request reception from Kafka publishing.

### Negative

- Two independently deployed collector services introduce operational overhead.
- Shared libraries must be versioned and maintained carefully.
- In-memory queues do not survive collector process crashes.
- Pull replica coordination introduces additional complexity.
- Kafka partitioning and capacity must be designed to handle uneven source traffic.

## Future Considerations

The following decisions are intentionally postponed:

- Pull Collector device assignment, leases and ownership recovery.
- Push Collector queue limits, HTTP backpressure responses and retry policies.
- Kafka partition count, partition key and replication configuration.
- Durable disk-backed buffering.
- Source-specific protocol adapters.
- Source-health monitoring and collection failure events.

These decisions will be made when their implementation requirements and workload characteristics are better understood.