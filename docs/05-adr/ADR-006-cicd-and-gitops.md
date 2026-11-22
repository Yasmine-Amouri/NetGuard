# ADR-006: CI/CD and GitOps Strategy

- **Status:** Accepted
- **Date:** 2026-11-19

## Context

NetGuard consists of multiple independently deployable services that require automated building, testing, security validation, and deployment.

The platform needs a CI/CD architecture that supports:

- Automated builds and tests triggered by code changes.
- Security scanning of source code, dependencies, and container images.
- Automated Docker image creation and publishing.
- Controlled deployments to Kubernetes.
- Consistent deployment configurations across local and cloud environments.
- Traceability and reproducibility of application deployments.

The architecture should also separate application integration and validation from deployment management.

## Decision Drivers

- Automate software integration, testing, and delivery.
- Separate CI responsibilities from Kubernetes deployment responsibilities.
- Maintain a clear source of truth for the desired deployment state.
- Support independent pipelines for different services.
- Provide flexibility for complex automation workflows.
- Integrate with Kubernetes and the GitOps approach.

## Alternatives Considered

### A. GitHub Actions for CI/CD

Use GitHub Actions to build, test, scan, publish images, and deploy applications.

**Advantages:**
- Native integration with GitHub repositories.
- Simple workflow configuration using YAML.
- Supports automated builds, tests, and deployments.
- Reduces the need to operate a dedicated CI server.

**Disadvantages:**
- CI and deployment responsibilities would initially be handled by the same platform.
- Does not provide the same dedicated GitOps reconciliation model as Argo CD.
- Does not provide the same experience of operating and maintaining a dedicated CI server.

### B. Jenkins + Argo CD

Use Jenkins for Continuous Integration and Argo CD for GitOps-based Continuous Delivery.

**Advantages:**
- Clear separation between CI and CD responsibilities.
- Jenkins provides customizable pipelines, build agents, and extensive plugin integrations.
- Argo CD continuously monitors Git-based deployment configurations and synchronizes Kubernetes with the desired state.
- Supports automated deployment reconciliation and drift detection.
- Jenkins does not need direct access to the Kubernetes cluster for deployment operations.

**Disadvantages:**
- Requires maintaining an additional CI server.
- Jenkins configuration, plugins, credentials, and build agents introduce operational complexity.
- Requires coordination between Jenkins pipelines and the GitOps repository.

### C. GitHub Actions + Argo CD

Use GitHub Actions for CI and Argo CD for GitOps-based CD.

**Advantages:**
- Separates CI and CD responsibilities.
- Native GitHub integration.
- Argo CD provides Kubernetes deployment reconciliation.
- Less CI infrastructure to maintain when using GitHub-hosted runners.

**Disadvantages:**
- Provides less experience operating a dedicated CI server.
- Requires configuring the interaction between GitHub Actions and the GitOps repository.

## Decision

We choose **Jenkins for Continuous Integration and Argo CD for GitOps-based Continuous Delivery**.

Their responsibilities will be:

**Jenkins — CI**
- Trigger pipelines when relevant source code changes are pushed to GitHub.
- Build and test application services.
- Perform automated security and quality checks.
- Build Docker images.
- Publish versioned images to GitHub Container Registry (GHCR).
- Update the corresponding image version in the GitOps repository.

**Argo CD — GitOps CD**
- Monitor the GitOps repository for deployment configuration changes.
- Compare the desired deployment state declared in Git with the actual Kubernetes cluster state.
- Synchronize Kubernetes resources with the desired state.
- Detect configuration drift and optionally restore the desired state through automated self-healing.

The GitOps repository will serve as the source of truth for Kubernetes deployment configurations.

Jenkins will not directly deploy application resources to Kubernetes. Argo CD will manage that responsibility.

## Consequences

**Positive:**
- Clear separation between CI and CD.
- Automated software validation and image publishing.
- Declarative and version-controlled deployment configurations.
- Improved deployment traceability and reproducibility.
- Kubernetes state can be continuously reconciled with Git.
- Practical experience with Jenkins administration and Kubernetes GitOps workflows.

**Negative:**
- Additional infrastructure and maintenance requirements for Jenkins and Argo CD.
- More configuration and integration between pipelines and repositories.
- CI failures, registry failures, or GitOps synchronization failures require separate monitoring and troubleshooting.
- Jenkins requires appropriate credential and permission management.

## Future Considerations

- Use Jenkins agents to execute pipeline workloads.
- Implement path-based pipeline triggers for independently deployable services.
- Integrate automated tests and security scanning into CI.
- Use immutable, versioned container image tags.
- Avoid using mutable tags such as `latest` for production deployments.
- Configure Argo CD synchronization policies and self-healing.
- Secure communication and credentials between Jenkins, GitHub, GHCR, and the GitOps repository.
- Evaluate Jenkins and Argo CD availability and recovery strategies.