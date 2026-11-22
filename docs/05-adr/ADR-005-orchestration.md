# ADR-005: Container Orchestration Strategy

- **Status:** Accepted
- **Date:** 2026-11-19

## Context

NetGuard consists of multiple independently deployable services, including the telemetry collector, stream processor, ML inference service, persistence worker, incident worker, and platform API.

As the platform grows, it requires an orchestration solution capable of:

- Managing multiple containerized services.
- Automatically recovering from application failures.
- Performing application health checks.
- Scaling services independently.
- Supporting reliable deployments and service discovery.
- Running locally and in a cloud environment.

## Alternatives Considered

### A. Docker Compose

Use Docker Compose to manage the application's containers.

**Advantages:**
- Simple configuration and local development.
- Supports health checks and restart policies.
- Suitable for running multiple containers on a single machine.

**Disadvantages:**
- Does not provide full cluster orchestration.
- Limited capabilities for independently scaling and managing workloads across multiple machines.
- An unhealthy container is not automatically restarted by Compose's standard restart policy solely because its health check fails.

### B. Kubernetes

Use Kubernetes to orchestrate application containers across a cluster.

**Advantages:**
- Automatic container recovery and workload management.
- Supports application health checks through startup, liveness, and readiness probes.
- Supports independent scaling of services.
- Provides service discovery, load balancing, and rolling deployments.
- Offers a large cloud-native ecosystem.

**Disadvantages:**
- Additional configuration and operational complexity.
- Requires cluster resources even for local development.

### C. HashiCorp Nomad

Use Nomad as the workload orchestrator.

**Advantages:**
- Supports container orchestration and workload scheduling.
- Provides health checks and automatic task recovery.
- Generally has a simpler operational model than Kubernetes.

**Disadvantages:**
- Smaller ecosystem for some Kubernetes-oriented cloud-native integrations.

## Decision

We choose **Kubernetes as the primary orchestration platform** for NetGuard.

The development and deployment environments will be:

- **Local development:** Kind, providing a local Kubernetes cluster using Docker containers as nodes.
- **Cloud deployment:** Azure Kubernetes Service (AKS), providing a managed Kubernetes environment on Azure.

Docker will continue to be used for building and packaging application images. Kubernetes will manage the deployment and lifecycle of those containers.

Helm will be used to package and manage Kubernetes application deployments.

## Consequences

**Positive:**
- Consistent orchestration approach between local development and cloud deployment.
- Practical experience with Kubernetes and cloud-native technologies.
- Automatic recovery through Kubernetes controllers and health probes.
- Independent scaling of application services.
- Improved deployment management through rolling updates and declarative configuration.

**Negative:**
- Local Kubernetes clusters consume machine resources.
- AKS and associated Azure resources introduce cloud infrastructure costs.
- Some environment-specific configuration will still be required for networking, storage, and cloud integrations.

## Future Considerations

- Configure resource requests and limits for each service.
- Implement startup, liveness, and readiness probes where appropriate.
- Introduce Horizontal Pod Autoscaling when workload metrics and scaling requirements are established.
- Use Helm for reusable deployment configurations.
- Evaluate Kubernetes security through RBAC, NetworkPolicies, and admission policies.