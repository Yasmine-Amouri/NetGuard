# NetGuard — System Architecture

## 1. Overview

NetGuard follows a hybrid, event-driven architecture composed of independently deployable services, orchestrated by Kubernetes. Kafka provides asynchronous communication between telemetry-processing components, while HTTP/REST handles synchronous application requests.

## 2. Main Components

### Telemetry Collection

Two independently deployable services collect telemetry:

- **Pull Collector:** Periodically requests measurements from network devices. Multiple replicas distribute device assignments to avoid duplicate polling.
- **Push Collector:** Receives measurements from external devices and agents. Multiple replicas operate behind a Kubernetes Service to distribute incoming requests.

Both collectors share common libraries for event normalization, validation and Kafka publishing. They transform source-specific measurements into a canonical telemetry event format and publish them to the same Kafka topic: `telemetry.raw`.

### Stream Processing

- The Stream Processor consumes normalized telemetry from `telemetry.raw` and performs source-independent validation, platform-level metadata enrichment, and event-level transformations before publishing individual observations to `telemetry.processed`. 
- Streaming aggregation is postponed; individual measurements are persisted, with aggregation handled through database queries when needed. 
- Invalid or unprocessable events are routed to `telemetry.dlq`, while temporary failures are handled through bounded retries. 
- Each replica maintains an independent in-memory metadata cache, initialized through a consistent PostgreSQL snapshot coordinated with a CDC position. 
- Replicas independently consume metadata changes to keep their caches synchronized. 
- Existing replicas continue processing with their last known metadata during temporary PostgreSQL outages, marking potentially stale enrichment, while new replicas must initialize their cache before processing telemetry. 
- Once metadata is synchronized again, corrections are published through `telemetry.enrichment.updated`, allowing downstream consumers to update stored metadata and recalculate incident priorities without treating corrections as new measurements. 
- The processor uses at-least-once delivery, Kafka producer idempotence, and application-level idempotency based on stable event identifiers to protect against duplicate processing and side effects. Both measurement and ingestion timestamps are preserved, and late observations are retained rather than discarded.


Detailed architectural decisions are documented in `docs/05-adr/`.