# ADR-004: Database Choice

- **Status:** Accepted
- **Date:** 2026-11-18

## Context

NetGuard handles two main categories of data:

1. **Operational data:** users, networks, network links, incidents, and model metadata. These entities have relationships and require transactional consistency.
2. **Telemetry data:** timestamped network measurements such as RTT, jitter, and packet loss. These records are continuously generated and frequently queried over time ranges.

The platform needs a database solution that supports relational operations while efficiently storing and querying time-series data.

## Decision Drivers

- Support relational data and transactions.
- Efficiently store and query large volumes of time-series telemetry.
- Avoid unnecessary infrastructure complexity during the initial development phase.
- Allow future evolution as data volume and analytical requirements grow.

## Alternatives Considered

### A. PostgreSQL alone

Use PostgreSQL for both operational data and telemetry.

**Advantages:**
- Simple architecture and deployment.
- Strong relational capabilities and transactional consistency.
- Familiar SQL interface.

**Disadvantages:**
- Large telemetry workloads may require additional optimization and partitioning.
- Time-series-specific operations may become less efficient as data volume grows.

### B. PostgreSQL with TimescaleDB extension

Use PostgreSQL with the TimescaleDB extension to support both operational and time-series data within one database instance.

**Advantages:**
- Retains PostgreSQL's relational capabilities and SQL interface.
- Provides time-series optimizations, including hypertables and time-based partitioning.
- Avoids introducing a separate database service initially.

**Disadvantages:**
- Adds extension-specific configuration and maintenance.
- Requires evaluating compatibility and deployment constraints.

### C. PostgreSQL + ClickHouse

Use PostgreSQL for operational data and ClickHouse for large-scale telemetry analytics.

**Advantages:**
- Separates transactional workloads from analytical workloads.
- ClickHouse is optimized for large-scale analytical queries and aggregations.

**Disadvantages:**
- Introduces another database service.
- Requires data synchronization and additional operational management.
- Adds complexity that may not be justified initially.

## Decision

We choose **PostgreSQL with the TimescaleDB extension**, deployed as a single database instance initially.

Responsibilities:

- PostgreSQL handles users, networks, network links, incidents, and model metadata.
- TimescaleDB capabilities optimize the storage and querying of telemetry measurements.

ClickHouse may be considered in the future if measured telemetry volume and analytical workloads justify introducing a dedicated analytical database.

## Consequences

**Positive:**
- A unified database solution for operational and telemetry data.
- Relational consistency for entities such as networks, links, and incidents.
- Time-series optimizations for continuous telemetry ingestion and historical queries.
- Reduced infrastructure complexity compared with operating multiple database services.

**Negative:**
- Additional configuration and maintenance associated with the TimescaleDB extension.
- Database performance must be monitored as telemetry volume increases.
- Future adoption of ClickHouse would require an additional data pipeline or synchronization mechanism.

## Future Considerations

Database scaling decisions will be based on measured workload characteristics, including:

- Telemetry ingestion rate.
- Storage growth.
- Query latency for historical aggregations.
- Database resource consumption.

ClickHouse will only be introduced if these measurements demonstrate a need for a dedicated analytical database.