# ADR-009: Stream Processor Architecture

**Status:** Accepted  
**Date:** 2026-11-22

## 1. Context

The Stream Processor receives normalized telemetry from the Pull and Push Collectors through Kafka. It must perform common validation, metadata enrichment and event-level transformations while remaining independent of source-specific formats.

It must also handle temporary infrastructure failures, duplicate message delivery, retries, metadata synchronization and late observations without compromising downstream processing reliability.

## 2. Decision

### 2.1. Processing responsibilities

The Stream Processor will:

- Consume normalized events from `telemetry.raw`.
- Perform source-independent validation.
- Enrich observations with platform-level metadata when available.
- Perform event-level transformations.
- Publish validated individual observations to `telemetry.processed`.

Streaming aggregation is postponed. Individual observations are persisted, and aggregation can initially be performed through database queries.

The processor preserves two timestamps:

- `event_timestamp`: when the measurement actually occurred.
- `ingestion_timestamp`: when NetGuard received the measurement.

Late observations are retained for historical analysis rather than discarded because newer observations have already arrived. They do not automatically generate real-time alerts.

### 2.2. Metadata enrichment and synchronization

Each Stream Processor replica maintains its own in-memory metadata cache.

**CDC technology selection**

Debezium with its PostgreSQL connector is selected to capture metadata changes from PostgreSQL's Write-Ahead Log (WAL) and publish them to Kafka.

Alternatives considered:

- **Custom PostgreSQL logical replication consumer:** Provides control but requires implementing snapshot coordination, offset tracking and recovery mechanisms.
- **Periodic database polling:** Simpler initially, but introduces repeated queries, detection delays and additional change-tracking logic.

Debezium is preferred because it provides an established CDC mechanism and integrates with our existing Kafka infrastructure.

**Cache initialization and synchronization**

- Each replica initializes its cache using a consistent PostgreSQL snapshot coordinated with a CDC position.
- Changes occurring after the snapshot position are applied to the cache.
- Every replica independently receives all metadata changes and maintains its own consumption progress.
- Replicas must not share a Kafka consumer group for metadata changes, because that would distribute events between them instead of delivering every update to every replica.
- Each replica uses an independent consumer group or equivalent independent subscription and offset tracking.

**Recovery behavior**

Two situations are distinguished:

1. **Temporary disconnection:** If the replica retains its in-memory cache, it resumes consuming from its last successfully processed CDC position, provided the required changes remain available.
2. **Process restart or cache loss:** The replica rebuilds its cache from a consistent snapshot and resumes CDC from the corresponding coordinated position. It must not simply resume from its previous offset with an empty cache.

If the required WAL changes are no longer available, the cache must be reinitialized through a fresh snapshot.

**PostgreSQL outage policy**

Existing replicas continue processing telemetry using their last known metadata during temporary PostgreSQL or CDC outages. Enrichment is explicitly marked as potentially stale.

A newly started replica must initialize its metadata cache before processing telemetry.

Once metadata synchronization is restored, the processor publishes corrections through a separate `telemetry.enrichment.updated` topic instead of republishing the original telemetry as a new observation.

### 2.3. Metadata correction handling

Correction events identify the original `event_id` and include a metadata version.

Downstream consumers handle corrections independently:

- **Persistence Worker:** Updates the associated stored metadata.
- **Incident Worker:** Recalculates current incident priority when relevant metadata changes.
- **ML Worker:** Does not repeat inference solely because metadata was corrected.

Historical metadata at detection time is preserved separately from current metadata. This allows NetGuard to retain the context in which an anomaly originally occurred while keeping current incident information accurate.

### 2.4. Error handling and retries

The processor distinguishes permanent errors from temporary failures.

**Permanent or unprocessable events**

Examples:
- Malformed payloads.
- Missing required fields.
- Unknown link identifiers when authoritative metadata is available and confirms that they are invalid.

These events are routed to `telemetry.dlq` with their original payload and error information, allowing investigation and potential replay.

**Temporary failures**

Examples:
- Temporary database or dependency unavailability.
- Other recoverable processing failures.

These failures receive bounded retries using delays such as exponential backoff.

If immediate retries are exhausted, the event can be transferred to a dedicated retry topic for later processing. Events that remain unprocessable after the retry policy can eventually be routed to the DLQ.

A failure affecting one event must not unnecessarily block unrelated valid events in the same batch.

The processor considers responsibility transferred only after the destination Kafka topic acknowledges publication. It can then commit the corresponding source offset.

### 2.5. Delivery guarantees and idempotency

The processor follows an **at-least-once delivery model**. Consequently, an event may be delivered or published more than once.

Two forms of idempotency are used.

**Kafka producer-level idempotency**

Kafka producer idempotence protects against certain duplicate writes caused by protocol-level producer retries, such as when a broker stores a message but its acknowledgement is lost.

Kafka uses producer identity and sequence numbers to recognize retransmissions.

However, this mechanism does not cover every application-level duplicate. Independently sending the same logical event again, or republishing it after an application restart, is not necessarily recognized as a protocol-level retransmission.

**Application-level idempotency**

Every telemetry observation has a stable, unique `event_id`, preserved across processing stages, retries and replays.

Consumers protect their side effects using this identifier:

- **Persistence Worker:** Enforces uniqueness through a database constraint on `event_id`.
- **ML Worker:** Inference may be repeated when necessary, but downstream anomaly publication and other side effects must be protected against duplicates.
- **Incident Worker:** Prevents multiple incidents from being created for the same logical anomaly.

### 2.6. Distinguishing retries from duplicate deliveries

These are different situations.

**Intentional retry**

An event fails because of a temporary problem and is deliberately sent for another processing attempt.

For example, E123 fails three times before successfully reaching `telemetry.processed`.

**Duplicate delivery**

An event is delivered again because of at-least-once processing.

For example, the processor successfully publishes E123 to `telemetry.processed` but crashes before committing its source offset. After restarting, it consumes E123 again and may publish it a second time.

The same `event_id` can legitimately appear in different processing stages, such as `telemetry.raw`, `telemetry.retry` and `telemetry.processed`.

Therefore, deduplication must be scoped to the relevant processing operation or business side effect, not implemented as a global rejection of every previously seen `event_id`.

### 2.7. Publishing and committing offsets

Publishing an output event and committing its source offset are separate operations.

A crash between these operations can cause the input event to be delivered again and the output to be published more than once.

The initial architecture accepts this at-least-once behavior and relies on stable event identifiers and application-level idempotency.

Kafka transactions, which can atomically combine Kafka output publication and consumed-offset commits, may be considered later if operational requirements justify the additional complexity. They are not required initially.

## 3. Consequences

**Benefits**

- Source-independent processing and consistent events for downstream consumers.
- Independent metadata caches avoid querying PostgreSQL for every observation.
- CDC propagates metadata changes without manually updating every cache.
- Existing replicas can continue processing during temporary PostgreSQL outages.
- At-least-once delivery provides recovery without requiring distributed exactly-once transactions.
- Producer-level and application-level idempotency protect against different classes of duplicates.
- Separate correction events prevent metadata updates from being mistaken for new measurements.
- Individual observations remain available for historical analysis and future aggregation.

**Trade-offs**

- Each replica must maintain and recover its own cache and CDC progress.
- Metadata can temporarily be stale during database or CDC outages.
- CDC retention and PostgreSQL WAL storage must be monitored.
- Retry topics, DLQ handling and replay require additional operational management.
- Consumers must implement idempotency at their side-effect boundaries.
- Metadata correction events introduce additional consistency requirements.
- Kafka buffering and consumer lag must be monitored during processing interruptions.
- Debezium introduces an additional infrastructure component to operate and monitor.

## 4. Alternatives considered

| Alternative | Reason not selected |
|---|---|
| Query PostgreSQL for every telemetry event | Introduces database load and latency. |
| Publish metadata updates manually through an application-managed Kafka topic | Requires every metadata-writing operation to reliably publish a corresponding event. |
| Custom PostgreSQL logical replication consumer | Requires implementing CDC synchronization and recovery ourselves. |
| Periodic database polling | Repeated queries and delayed change detection. |
| Stop all processing whenever PostgreSQL becomes unavailable | Stronger metadata freshness but lower processing availability. |
| Continue processing without enrichment and replay pending events later | Introduces an additional event lifecycle and recovery mechanism. |
| Kafka transactions for every processing operation | Additional complexity not currently justified by our at-least-once requirements. |

## 5. Future points to consider

The following are intentionally postponed and do not block the initial architecture:

- **CDC retention and recovery:** Define WAL retention limits, replication-slot monitoring, and snapshot reinitialization procedures.
- **Metadata versioning:** Define how consumers reject outdated metadata corrections and handle concurrent updates.
- **Latest-update optimization:** Consider retaining only the latest metadata update per link when reconstructing caches, rather than replaying every historical update. This is an optimization, not an initial requirement.
- **Non-idempotent CDC operations:** Introduce explicit idempotency or version-based protection if metadata processing evolves beyond simple assignments.
- **Retry policy:** Define retry limits, backoff parameters, retry-topic retention and replay procedures.
- **DLQ management:** Define investigation, correction and controlled replay workflows.
- **Kafka capacity:** Establish consumer-lag thresholds, backlog monitoring and scaling policies.
- **Observability:** Monitor processing latency, validation failures, duplicate deliveries, retry rates, DLQ volume, CDC lag and metadata-cache freshness.
- **Streaming aggregation:** Reconsider if real-time rolling statistics or sequence-based processing become necessary.
- **Kafka transactions:** Reevaluate if stronger Kafka-to-Kafka atomicity becomes an operational requirement.