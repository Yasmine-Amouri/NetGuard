# ADR-001: Event-Driven Architecture with Kafka

- **Status:** Accepted
- **Date:** 2026-11-16

## 1. Context

NetGuard collects network telemetry from historical datasets and a synthetic network simulator.

The collected measurements must be validated, processed, stored, and analyzed by different components.

Direct communication between the collector and processor could create tight coupling. If the processor becomes unavailable or cannot handle the incoming event rate, the collector may experience failed requests and require additional retry and buffering mechanisms.

Furthermore, multiple services will eventually need to consume the same telemetry independently.

We therefore need a communication architecture that supports asynchronous processing, temporary buffering, and independent consumers.

## 2. Decision Drivers

- Decouple telemetry producers from consumers.
- Handle temporary processing slowdowns and service unavailability.
- Allow multiple independent consumers to receive the same events.
- Support replaying retained events.
- Allow services to scale independently.

## 3. Alternatives Considered

### Alternative A: Direct HTTP communication

The collector sends HTTP requests directly to the processor.

**Advantages:**
- Simple to implement and understand.
- No additional messaging infrastructure.
- Suitable for synchronous request-response interactions.

**Disadvantages:**
- Producer and consumer availability are more tightly coupled.
- Requires additional mechanisms for retries and buffering.
- Multiple consumers require additional communication logic.

### Alternative B: Kafka-based communication

The collector publishes events to Kafka topics, and consumers retrieve them asynchronously.

**Advantages:**
- Decouples producers and consumers.
- Buffers events during temporary processing slowdowns.
- Supports independent consumer groups.
- Allows consumers to replay retained events.

**Disadvantages:**
- Introduces additional infrastructure.
- Requires topic, partition, offset, and consumer-group management.
- Increases operational and debugging complexity.

### Alternative C: Shared database as a message queue

The collector inserts events into a database, and consumers retrieve pending records.

**Advantages:**
- Uses familiar database technology.
- Avoids introducing a dedicated message broker.

**Disadvantages:**
- Requires application-level coordination and processing-state management.
- Concurrent consumers require careful handling.
- Replaying and distributing events to independent consumers requires additional logic.

## 4. Decision

We adopt a hybrid communication architecture:

- **Kafka** will be used for asynchronous telemetry and event distribution.
- **HTTP/REST** will be used for synchronous API communication, particularly between the dashboard and platform API.

Kafka will initially handle telemetry ingestion and distribution between processing components.

## 5. Consequences

### Positive consequences
- Services become less dependent on each other's availability.
- Telemetry can be buffered during temporary processing slowdowns.
- Multiple services can consume the same event stream independently.
- Consumers can replay retained events when necessary.
- Processing components can scale independently.

### Negative consequences
- Kafka introduces infrastructure and operational complexity.
- Developers must understand consumer groups, partitions, offsets, and message delivery semantics.
- Additional monitoring and configuration are required.

## 6. Reliability Considerations

The initial implementation will use at-least-once message processing semantics.

Consumers must therefore be designed to handle duplicate events safely through idempotent processing.

Kafka retention policies, producer acknowledgements, consumer offset management, retries, and dead-letter handling must be configured and tested.