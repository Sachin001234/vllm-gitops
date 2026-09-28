# 🔄 vLLM GitOps Repository

A dedicated **GitOps repository for managing the Kubernetes desired state of the vLLM inference platform**.

This repository is separate from the `vllm-platform` repository and contains the Kubernetes manifests, Kustomize overlays, monitoring configuration, and Argo CD-related resources used to deploy and operate the platform.

The repository follows the GitOps principle of keeping **Kubernetes desired state in Git** and allowing **Argo CD to continuously reconcile that state with the Kubernetes cluster**.

---

## 📌 GitOps Repository Overview

The repository is responsible for managing the Kubernetes side of the vLLM platform.

```text
vllm-platform
      │
      │ Container Image
      ▼
     GHCR
      │
      │ Image Reference
      ▼
vllm-gitops
      │
      ▼
   Argo CD
      │
      ▼
 Kubernetes
      │
      ▼
 vLLM Workload
```

The repository contains environment-specific Kubernetes configuration using **Kustomize**:

```text
                    vllm-gitops
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
            base      overlays    monitoring
                         │
                    ┌────┴────┐
                    ▼         ▼
                   dev       prod
```

### 🔹 Main Responsibilities

The GitOps repository manages:

- Kubernetes Deployments
- Kubernetes Services
- NGINX Ingress
- Kustomize configuration
- Development environment configuration
- Production-oriented configuration
- GPU scheduling configuration
- Kubernetes Secret references
- Prometheus ServiceMonitor
- Prometheus alert rules
- Alertmanager Slack configuration
- Argo CD deployment configuration

### 🔹 GitOps Principle

The repository acts as the **source of truth for Kubernetes desired state**.

```text
Git Repository
      │
      │ Desired State
      ▼
    Argo CD
      │
      │ Reconciliation
      ▼
 Kubernetes Cluster
      │
      │ Actual State
      ▼
   vLLM Platform
```

Changes to Kubernetes configuration are made through Git rather than manually modifying the cluster.

Argo CD monitors the repository and synchronizes the Kubernetes cluster with the desired state stored here.

### 🔹 Repository Separation

The project intentionally uses two repositories:

| Repository | Responsibility |
|---|---|
| `vllm-platform` | vLLM configuration, Docker, tests, and CI/CD |
| `vllm-gitops` | Kubernetes desired state and GitOps configuration |

The CI/CD pipeline from `vllm-platform` automatically updates the container image reference in this repository after a successful image build.

Argo CD then detects the Git change and synchronizes the updated desired state to Kubernetes.

# 🏗️ Architecture

The `vllm-gitops` repository is designed around the **GitOps deployment model**, where Git stores the desired Kubernetes state and **Argo CD continuously reconciles that state with the Kubernetes cluster**.

---

## High-Level Architecture

```text
                         Developer
                             │
                             │ Code / Configuration
                             ▼
                    ┌─────────────────┐
                    │  vllm-platform  │
                    │   GitHub Repo   │
                    └────────┬────────┘
                             │
                             │ GitHub Actions
                             │
                             ▼
                    ┌─────────────────┐
                    │ Container Image │
                    │      GHCR       │
                    └────────┬────────┘
                             │
                             │ Image Reference
                             ▼
                    ┌─────────────────┐
                    │   vllm-gitops   │
                    │   GitHub Repo   │
                    └────────┬────────┘
                             │
                             │ GitOps
                             ▼
                    ┌─────────────────┐
                    │     Argo CD     │
                    └────────┬────────┘
                             │
                             │ Reconciliation
                             ▼
              ┌──────────────────────────────┐
              │      Kubernetes Cluster      │
              │                              │
              │  ┌────────────────────────┐  │
              │  │      vLLM Deployment   │  │
              │  └───────────┬────────────┘  │
              │              │               │
              │              ▼               │
              │       vLLM Inference API     │
              │                              │
              │  Service → Ingress           │
              │  Prometheus → Grafana        │
              │  Alertmanager → Slack        │
              └──────────────────────────────┘
```

---

## Two-Repository Architecture

The project separates application delivery from Kubernetes deployment management.

### `vllm-platform`

Responsible for:

```text
Application / Platform Side
        │
        ├── vLLM configuration
        ├── Dockerfile
        ├── Docker Compose
        ├── Automated tests
        ├── GitHub Actions
        └── Container image build
```

The CI pipeline builds the container image and publishes it to **GitHub Container Registry (GHCR)**.

### `vllm-gitops`

Responsible for:

```text
Kubernetes / GitOps Side
        │
        ├── Kubernetes manifests
        ├── Kustomize
        ├── Environment overlays
        ├── Ingress
        ├── Monitoring
        ├── Alerting
        └── Argo CD configuration
```

The GitOps repository does **not build the application image**.

Instead, it stores the desired Kubernetes configuration and the image reference that Kubernetes should deploy.

---

## CI → GitOps → Kubernetes Flow

A successful change follows this flow:

```text
Developer
    │
    ▼
vllm-platform
    │
    ▼
GitHub Actions
    │
    ├── Run tests
    ├── Build Docker image
    └── Push image to GHCR
    │
    ▼
Update vllm-gitops
    │
    └── Update image tag
            │
            ▼
        Argo CD
            │
            ├── Detect Git change
            ├── Compare desired state
            └── Synchronize cluster
                    │
                    ▼
               Kubernetes
                    │
                    ▼
                  vLLM
```

The image is referenced using an immutable Git commit SHA rather than relying only on a mutable tag such as `latest`.

Example:

```yaml
image: ghcr.io/sachin001234/vllm-platform:<commit-sha>
```

This makes deployments easier to trace and roll back.

---

## Kustomize Architecture

The repository uses **Kustomize** to separate common Kubernetes resources from environment-specific configuration.

```text
                    base/
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   Deployment      Service      Ingress
        │
        │
        ▼
   Common Configuration
        │
        ├───────────────┐
        │               │
        ▼               ▼
   overlays/dev    overlays/prod
        │               │
        ▼               ▼
   Development      Production
   Configuration    Configuration
```

The `base` directory contains reusable Kubernetes resources.

The environment overlays modify the base configuration without duplicating the entire application definition.

---

## Development Architecture

The development environment is managed through:

```text
overlays/dev/
```

It combines the common base with development-specific configuration.

```text
             overlays/dev
                  │
          ┌───────┼────────┐
          ▼       ▼        ▼
       namespace  base   monitoring
          │       │        │
          │       │        ├── ServiceMonitor
          │       │        └── Alert Rules
          │       │
          │       ├── Deployment
          │       ├── Service
          │       └── Ingress
          │
          └── Development-specific patches
```

The development overlay also contains the runtime configuration used for the local vLLM environment.

---

## Production-Oriented Architecture

The production overlay is located at:

```text
overlays/prod/
```

It adds production-oriented Kubernetes configuration such as:

```text
overlays/prod/
├── gpu-patch.yaml
├── replica-patch.yaml
├── resources-patch.yaml
├── rolling-update-patch.yaml
├── topology-patch.yaml
├── pdb.yaml
├── secret-env-patch.yaml
├── namespace.yaml
└── kustomization.yaml
```

The production configuration is designed for a Kubernetes cluster with NVIDIA GPU nodes.

```text
              Kubernetes Cluster
                     │
             NVIDIA GPU Node
                     │
              ┌──────▼──────┐
              │ vLLM Pod    │
              │             │
              │ GPU: 1      │
              │ CPU / RAM   │
              └─────────────┘
```

The production overlay currently defines **1 vLLM replica** and requests one NVIDIA GPU.

It also contains configuration for:

- CPU and memory resources
- GPU scheduling
- rolling updates
- topology spreading
- PodDisruptionBudget
- secrets
- namespace isolation

These settings are production-oriented configuration; the local Docker Desktop Kubernetes cluster does not provide the required NVIDIA GPU.

---

## Monitoring Architecture

Monitoring is also managed through the GitOps repository.

```text
                    vLLM
                     │
                     │ /metrics
                     ▼
                ServiceMonitor
                     │
                     ▼
                 Prometheus
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
       Grafana              Alert Rules
                                  │
                                  ▼
                             Alertmanager
                                  │
                                  ▼
                               Slack
```

The repository contains monitoring configuration for:

- vLLM metrics collection
- Prometheus ServiceMonitor
- Prometheus alert rules
- Alertmanager Slack integration

This allows monitoring configuration to be version-controlled alongside the Kubernetes deployment configuration.

---

## GitOps Reconciliation

Argo CD continuously compares:

```text
Git Desired State
        │
        │ compare
        ▼
Kubernetes Actual State
```

If the states differ, Argo CD can synchronize the cluster.

Example:

```text
Git:
image: vllm-platform:ABC123

Kubernetes:
image: vllm-platform:OLD456
```

Argo CD detects the difference:

```text
             Difference Detected
                     │
                     ▼
              Argo CD Sync
                     │
                     ▼
             Kubernetes Update
                     │
                     ▼
              Desired State
                Restored
```

This also provides self-healing behavior when automated synchronization and self-heal are enabled.

---

## Rollback Architecture

Because deployment state is stored in Git, rollback can be performed through Git history.

```text
Current Git Commit
        │
        ▼
   Deployment
        │
        │ Problem
        ▼
    git revert
        │
        ▼
Previous Known-Good State
        │
        ▼
     Argo CD
        │
        ▼
 Kubernetes
        │
        ▼
Previous Deployment State
```

This approach was tested in the project using a controlled invalid image deployment.

The invalid GitOps change caused the expected deployment failure, after which the Git commit was reverted and Argo CD restored the previous image reference.

---

## Complete GitOps Architecture

```text
                         ┌──────────────────┐
                         │    Developer     │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │ vllm-platform    │
                         │ GitHub Repository│
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │ GitHub Actions   │
                         │                  │
                         │ Test             │
                         │ Build            │
                         │ Push             │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │      GHCR        │
                         │ Container Image  │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │   vllm-gitops    │
                         │ GitHub Repository │
                         │                  │
                         │ Base             │
                         │ Dev Overlay      │
                         │ Prod Overlay     │
                         │ Monitoring       │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │     Argo CD      │
                         │                  │
                         │ Sync             │
                         │ Self-Heal        │
                         │ Rollback         │
                         └────────┬─────────┘
                                  │
                                  ▼
                  ┌─────────────────────────────┐
                  │       Kubernetes            │
                  │                             │
                  │   ┌─────────────────────┐   │
                  │   │       vLLM          │   │
                  │   └─────────┬───────────┘   │
                  │             │               │
                  │      Service / Ingress      │
                  │             │               │
                  │      ┌──────▼──────┐        │
                  │      │ vLLM API    │        │
                  │      └─────────────┘        │
                  │                             │
                  │   Prometheus → Grafana      │
                  │        │                    │
                  │        ▼                    │
                  │   Alertmanager → Slack      │
                  └─────────────────────────────┘
```

The architecture separates **application delivery**, **container image management**, **Kubernetes desired state**, **deployment reconciliation**, and **observability**, creating a complete GitOps workflow for the vLLM platform.

# 📁 Repository Structure

The `vllm-gitops` repository contains the Kubernetes desired state of the vLLM platform.

The repository is organized using **Kustomize**, separating reusable Kubernetes resources from environment-specific configuration.

---

## Complete Repository Structure

```text
vllm-gitops/
│
├── .git/
│
├── README.md
│
├── argocd/
│
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── kustomization.yaml
│
├── monitoring/
│   ├── kustomization.yaml
│   └── vllm-slack.yaml
│
└── overlays/
    │
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   ├── secret-env-patch.yaml
    │   ├── vllm-runtime-patch.yaml
    │   ├── vllm-servicemonitor.yaml
    │   │
    │   └── monitoring/
    │       └── vllm-alerts.yaml
    │
    └── prod/
        ├── kustomization.yaml
        ├── namespace.yaml
        ├── gpu-patch.yaml
        ├── replica-patch.yaml
        ├── resources-patch.yaml
        ├── rolling-update-patch.yaml
        ├── topology-patch.yaml
        ├── secret-env-patch.yaml
        └── pdb.yaml
```

---

## `base/`

The `base` directory contains the **common Kubernetes configuration** shared by environments.

```text
base/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
└── kustomization.yaml
```

### `deployment.yaml`

Defines the vLLM Kubernetes Deployment.

It contains common configuration such as:

- Deployment metadata
- vLLM container
- Container port
- vLLM image reference
- Health probes
- Common runtime configuration

The environment overlays can modify this Deployment using Kustomize patches.

---

### `service.yaml`

Creates the Kubernetes Service used to expose the vLLM Deployment internally.

```text
vLLM Pods
    │
    ▼
 Kubernetes Service
    │
    ▼
   Port 8000
```

The Service uses the `app: vllm` label to select the vLLM Pods.

---

### `ingress.yaml`

Defines the Kubernetes Ingress resource.

The project uses the **NGINX Ingress Controller** to provide HTTP routing toward the vLLM Service.

```text
Client
  │
  ▼
NGINX Ingress
  │
  ▼
vLLM Service
  │
  ▼
vLLM Pod
```

---

### `kustomization.yaml`

Defines the resources that make up the common base.

```text
base/kustomization.yaml
        │
        ├── deployment.yaml
        ├── service.yaml
        └── ingress.yaml
```

Environment overlays reuse this base rather than duplicating these manifests.

---

# `overlays/dev/`

The `dev` overlay contains configuration specific to the development environment.

```text
overlays/dev/
├── kustomization.yaml
├── namespace.yaml
├── secret-env-patch.yaml
├── vllm-runtime-patch.yaml
├── vllm-servicemonitor.yaml
└── monitoring/
    └── vllm-alerts.yaml
```

### `kustomization.yaml`

Combines:

```text
Dev Overlay
     │
     ├── Namespace
     ├── Base Resources
     ├── Secret Patch
     ├── Runtime Patch
     ├── ServiceMonitor
     └── Alert Rules
```

The resulting Kubernetes configuration is deployed into the:

```text
vllm-dev
```

namespace.

---

### `namespace.yaml`

Creates the development namespace:

```text
vllm-dev
```

Keeping the development workload in its own namespace provides environment isolation.

---

### `secret-env-patch.yaml`

References the Kubernetes Secret used by the vLLM application.

The actual secret value is **not stored in Git**.

Instead, Kubernetes Secret references are stored in the GitOps configuration.

```text
GitOps Repository
       │
       │ Secret Reference
       ▼
Kubernetes Secret
       │
       ▼
vLLM Pod
```

---

### `vllm-runtime-patch.yaml`

Contains development-specific vLLM runtime configuration.

The local development environment uses:

```text
Model:
Qwen/Qwen2.5-1.5B-Instruct-AWQ

Quantization:
AWQ

GPU Memory Utilization:
0.75

Maximum Model Length:
512

Execution:
enforce-eager
```

These settings were selected to make the model workable with the available local GPU resources during local vLLM testing.

---

### `vllm-servicemonitor.yaml`

Defines the Prometheus `ServiceMonitor` used to collect vLLM metrics.

```text
vLLM Service
     │
     │ /metrics
     ▼
ServiceMonitor
     │
     ▼
Prometheus
```

This connects the vLLM workload with the Prometheus monitoring stack.

---

### `monitoring/vllm-alerts.yaml`

Contains Prometheus alert rules for the development vLLM deployment.

One implemented alert detects when the deployment has no available replicas:

```text
VLLMPodDown
```

The alert can then be processed by Alertmanager and routed to Slack.

---

# `overlays/prod/`

The production overlay contains production-oriented Kubernetes configuration.

```text
overlays/prod/
├── kustomization.yaml
├── namespace.yaml
├── gpu-patch.yaml
├── replica-patch.yaml
├── resources-patch.yaml
├── rolling-update-patch.yaml
├── topology-patch.yaml
├── secret-env-patch.yaml
└── pdb.yaml
```

---

### `namespace.yaml`

Creates the production namespace:

```text
vllm-prod
```

This keeps production workloads isolated from development workloads.

---

### `gpu-patch.yaml`

Adds NVIDIA GPU scheduling requirements.

The production workload requests:

```yaml
nvidia.com/gpu: 1
```

and selects nodes labeled for NVIDIA GPUs.

```text
Kubernetes Scheduler
        │
        ▼
NVIDIA GPU Node
        │
        ▼
     vLLM Pod
```

This configuration is intended for a GPU-enabled Kubernetes cluster.

---

### `replica-patch.yaml`

Controls the number of vLLM replicas for the production environment.

The current production overlay is configured for:

```text
replicas: 1
```

Additional replicas can be configured when sufficient GPU capacity is available.

---

### `resources-patch.yaml`

Defines Kubernetes CPU and memory resource requests and limits.

The current production configuration includes:

```text
Requests:
  CPU:    2
  Memory: 8Gi

Limits:
  CPU:    4
  Memory: 12Gi
  GPU:    1
```

These values provide explicit resource requirements for scheduling the vLLM workload.

---

### `rolling-update-patch.yaml`

Defines the Deployment update strategy.

The production configuration uses:

```text
RollingUpdate
```

with controlled update behavior.

This allows Kubernetes to manage changes to the Deployment using a defined rollout strategy.

---

### `topology-patch.yaml`

Adds topology spread configuration using:

```text
kubernetes.io/hostname
```

The purpose is to provide a mechanism for distributing replicas across Kubernetes nodes when multiple replicas are deployed.

The current configuration has only one replica, so meaningful distribution is not achieved until additional replicas and suitable nodes are available.

---

### `secret-env-patch.yaml`

References production Kubernetes Secrets without storing their values inside Git.

```text
Git
 │
 └── Secret Reference
          │
          ▼
 Kubernetes Secret
          │
          ▼
      vLLM Pod
```

---

### `pdb.yaml`

Defines a Kubernetes `PodDisruptionBudget`.

The current configuration requires:

```text
minAvailable: 1
```

This helps protect the vLLM workload from voluntary disruptions when sufficient replicas are available.

---

# `monitoring/`

The `monitoring` directory contains shared monitoring and notification configuration.

```text
monitoring/
├── kustomization.yaml
└── vllm-slack.yaml
```

### `kustomization.yaml`

Defines the monitoring resources that should be managed through Kustomize.

---

### `vllm-slack.yaml`

Defines the Alertmanager configuration used to route vLLM alerts to Slack.

```text
Prometheus
    │
    ▼
Alertmanager
    │
    ▼
Slack
```

The Slack webhook itself is **not stored in Git**.

The configuration references a Kubernetes Secret containing the webhook.

---

# `argocd/`

The `argocd` directory is reserved for Argo CD-related Kubernetes resources.

Its purpose is to keep Argo CD application configuration separate from the application Deployment manifests.

Conceptually:

```text
argocd/
    │
    └── Argo CD Applications
             │
             ├── vllm-dev
             └── monitoring-config
```

Argo CD uses these Application definitions to determine:

- Which Git repository to monitor
- Which path to deploy
- Which Kubernetes cluster to target
- Which namespace to use
- Whether automated synchronization is enabled

---

# Environment Separation

The repository separates environments using Kustomize overlays.

```text
                    base
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
       overlays/dev         overlays/prod
          │                     │
          ▼                     ▼
      vllm-dev              vllm-prod
          │                     │
          ▼                     ▼
   Development              Production
   Configuration            Configuration
```

This provides:

- Shared configuration through `base`
- Environment-specific customization
- Reduced YAML duplication
- Clear separation between development and production
- Easier environment management
- Git-based change tracking

---

# Repository Design Principle

The repository follows a simple design rule:

```text
Base
 │
 ├── Common Kubernetes resources
 │
 └── Reusable configuration
          │
          ▼
       Overlays
          │
     ┌────┴────┐
     ▼         ▼
    Dev       Prod
     │         │
     ▼         ▼
Environment-specific configuration
```

This structure allows the same vLLM application definition to be reused across environments while keeping environment-specific requirements separate.

---

# GitOps Repository Responsibility

The final responsibility boundary is:

```text
vllm-platform
│
├── Application / platform code
├── Docker
├── Tests
├── CI/CD
└── Container image
          │
          ▼
       GHCR
          │
          ▼
vllm-gitops
│
├── Kubernetes desired state
├── Kustomize
├── Dev configuration
├── Prod configuration
├── Monitoring
├── Alerting
└── Argo CD
          │
          ▼
     Kubernetes
```

The `vllm-gitops` repository therefore acts as the **declarative deployment layer** of the overall vLLM DevOps platform.

# 🧩 Kustomize Base

The `base/` directory contains the **common Kubernetes resources** used by the vLLM platform.

Kustomize allows these resources to be defined once and then reused by different environments such as development and production.

```text
vllm-gitops/
│
└── base/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── kustomization.yaml
```

---

## Purpose of the Kustomize Base

The base provides the common Kubernetes architecture:

```text
                 Kustomize Base
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   Deployment       Service        Ingress
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                    vLLM
```

The base contains configuration that should remain common across environments.

Environment-specific settings are added later through overlays.

---

## `base/kustomization.yaml`

The base Kustomization file defines the resources that belong to the common vLLM application.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

This tells Kustomize to combine the three Kubernetes resources into one deployable configuration.

```text
kustomization.yaml
        │
        ├── deployment.yaml
        ├── service.yaml
        └── ingress.yaml
```

---

## `base/deployment.yaml`

The Deployment defines how the vLLM workload runs inside Kubernetes.

The common Deployment configuration includes:

- Deployment metadata
- vLLM container
- Container port
- Container image
- vLLM model configuration
- Liveness probe
- Readiness probe

Conceptually:

```text
Kubernetes Deployment
        │
        ▼
    ReplicaSet
        │
        ▼
      Pod
        │
        ▼
      vLLM
```

The Deployment uses the application label:

```yaml
app: vllm
```

This label is important because the Service and monitoring resources use it to identify the vLLM workload.

---

## Container Configuration

The vLLM container exposes port:

```text
8000
```

The application therefore follows:

```text
vLLM
 │
 └── HTTP API
       │
       └── Port 8000
```

The base Deployment provides the common container definition, while environment overlays can modify runtime arguments.

For example, the development overlay adds settings such as:

```text
--quantization awq
--gpu-memory-utilization 0.75
--max-model-len 512
--enforce-eager
```

This keeps environment-specific runtime configuration outside the common base.

---

## Health Checks

The Deployment defines Kubernetes health checks using the vLLM `/health` endpoint.

### Liveness Probe

```text
Liveness Probe
      │
      ▼
GET /health
      │
      ├── Healthy → Container continues running
      │
      └── Failed repeatedly → Kubernetes restarts container
```

The liveness probe helps Kubernetes determine whether the container is still functioning.

---

### Readiness Probe

```text
Readiness Probe
      │
      ▼
GET /health
      │
      ├── Ready → Pod can receive traffic
      │
      └── Not Ready → Pod removed from Service endpoints
```

The readiness probe prevents traffic from being sent to a Pod that is not ready to serve requests.

---

## Why Health Checks Are in the Base

Health checks are common application behavior and therefore belong in the shared base.

Both development and production environments can use the same basic health-check mechanism.

Environment-specific settings can still be modified through patches if required.

```text
                 Base
                  │
          /health probes
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
       Dev                 Prod
```

---

# `base/service.yaml`

The Service provides stable internal networking for the vLLM Pods.

```text
                    Kubernetes
                        │
                        ▼
                 ┌────────────┐
                 │   Service  │
                 │    vLLM    │
                 └─────┬──────┘
                       │
              ┌────────┴────────┐
              ▼                 ▼
           vLLM Pod          vLLM Pod
```

The Service selects Pods using:

```yaml
selector:
  app: vllm
```

The Service exposes:

```text
Service Port: 8000
Target Port: 8000
```

The traffic flow is therefore:

```text
Client
  │
  ▼
Ingress
  │
  ▼
vLLM Service :8000
  │
  ▼
vLLM Pod :8000
```

The Service uses:

```text
type: ClusterIP
```

This means it is primarily intended for internal Kubernetes networking.

---

# Why Use a Kubernetes Service?

Pods are not permanent network endpoints.

A Pod can be recreated during:

- Deployment updates
- failures
- scaling
- node changes
- self-healing

The Service provides a stable networking endpoint even when individual Pods change.

```text
Pod A
  │
  │ Pod replaced
  ▼
Pod B

       ▲
       │
       │
   Service
```

Applications and the Ingress do not need to know the individual Pod IP address.

---

# `base/ingress.yaml`

The Ingress defines external HTTP routing toward the vLLM Service.

The project uses:

```text
ingressClassName: nginx
```

The request path is:

```text
Client
  │
  ▼
NGINX Ingress Controller
  │
  ▼
vLLM Ingress
  │
  ▼
vLLM Service
  │
  ▼
vLLM Pod
```

The Ingress routes requests from:

```text
/
```

to:

```text
vLLM Service :8000
```

---

## Ingress Responsibility

The Ingress is responsible for HTTP routing.

It does not run vLLM itself.

```text
Ingress
   │
   └── Routing
        │
        ▼
Service
   │
   └── Load balancing
        │
        ▼
Pod
   │
   └── vLLM inference server
```

This separation keeps networking and application execution independent.

---

# Base → Overlay Model

The base is not intended to contain every environment-specific setting.

Instead:

```text
                    Base
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
   Deployment     Service      Ingress
        │
        ▼
   Common Config
        │
        ├──────────────────┐
        ▼                  ▼
   Dev Overlay        Prod Overlay
        │                  │
        ▼                  ▼
 Dev-specific         Production-
 configuration         oriented
                       configuration
```

This prevents duplication.

For example, the same `service.yaml` can be reused by both environments instead of maintaining separate copies.

---

# Kustomize Build Flow

When the development overlay is built, Kustomize starts with the base:

```text
overlays/dev/kustomization.yaml
              │
              ▼
        ../../base
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
  Deployment Service Ingress
       │
       ▼
Development Patches
       │
       ▼
Final Kubernetes Manifests
```

For production:

```text
overlays/prod/kustomization.yaml
              │
              ▼
        ../../base
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
  Deployment Service Ingress
       │
       ▼
Production Patches
       │
       ├── GPU
       ├── Resources
       ├── Replicas
       ├── Rolling Update
       ├── Topology
       └── PDB
              │
              ▼
Final Production Manifests
```

---

# Base Design Principle

The Kustomize base follows three main principles:

### 1. Reusability

Common Kubernetes resources are defined once.

### 2. Separation

Environment-specific configuration stays inside overlays.

### 3. Declarative Configuration

The desired Kubernetes state is stored as YAML in Git.

```text
              Git
               │
               ▼
          Kustomize Base
               │
       ┌───────┴───────┐
       ▼               ▼
      Dev             Prod
       │               │
       ▼               ▼
 Kubernetes Desired State
```

---

# Base Architecture Summary

```text
                     base/
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    deployment.yaml service.yaml ingress.yaml
          │            │            │
          │            │            │
          └────────────┼────────────┘
                       ▼
                vLLM Application
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      Workload      Networking     Routing
          │            │            │
          ▼            ▼            ▼
     Deployment     Service      NGINX
                                    │
                                    ▼
                                vLLM API
```

The `base/` directory therefore provides the **common Kubernetes foundation** of the vLLM platform, while the `dev` and `prod` overlays customize that foundation for their respective environments.

# 🧪 Development Environment

The `overlays/dev/` directory contains the Kubernetes configuration specific to the **development environment** of the vLLM platform.

The development environment reuses the common Kubernetes resources from `base/` and adds development-specific configuration through Kustomize patches and monitoring resources.

```text
overlays/dev/
│
├── kustomization.yaml
├── namespace.yaml
├── secret-env-patch.yaml
├── vllm-runtime-patch.yaml
├── vllm-servicemonitor.yaml
│
└── monitoring/
    └── vllm-alerts.yaml
```

---

## Development Architecture

The development environment follows this structure:

```text
                    overlays/dev
                         │
                         ▼
                    Kustomize
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
            base/              Dev Patches
              │                     │
              ├── Deployment       ├── Runtime
              ├── Service          └── Secrets
              └── Ingress
              │
              ▼
          vllm-dev Namespace
              │
              ▼
          vLLM Deployment
              │
              ▼
          vLLM Service
              │
              ▼
        NGINX Ingress
```

The development environment also integrates with the monitoring stack:

```text
vLLM
 │
 ├── /metrics
 │
 ▼
ServiceMonitor
 │
 ▼
Prometheus
 │
 ▼
Alert Rules
 │
 ▼
Alertmanager
 │
 ▼
Slack
```

---

# Development Namespace

The development environment uses a dedicated Kubernetes namespace:

```text
vllm-dev
```

This is defined in:

```text
overlays/dev/namespace.yaml
```

The namespace provides isolation between the development workload and other Kubernetes workloads.

```text
Kubernetes Cluster
│
├── vllm-dev
│   └── Development vLLM
│
├── monitoring
│   └── Monitoring Stack
│
└── argocd
    └── Argo CD
```

This separation makes it easier to manage, monitor, and troubleshoot the development deployment.

---

# Development Kustomization

The main development configuration is defined in:

```text
overlays/dev/kustomization.yaml
```

The overlay combines the common base with development-specific resources and patches.

Conceptually:

```text
overlays/dev/kustomization.yaml
              │
              ├── namespace.yaml
              │
              ├── ../../base
              │      ├── deployment.yaml
              │      ├── service.yaml
              │      └── ingress.yaml
              │
              ├── vllm-servicemonitor.yaml
              │
              ├── monitoring/vllm-alerts.yaml
              │
              ├── secret-env-patch.yaml
              │
              └── vllm-runtime-patch.yaml
```

This produces the final Kubernetes manifests for the development environment.

---

# Development Runtime Configuration

The file:

```text
vllm-runtime-patch.yaml
```

contains development-specific vLLM runtime arguments.

The current development configuration uses:

```text
Model:
Qwen/Qwen2.5-1.5B-Instruct-AWQ

Quantization:
AWQ

GPU Memory Utilization:
0.75

Maximum Model Length:
512

Execution Mode:
enforce-eager
```

The configuration is designed around the local hardware used during development.

The runtime configuration is effectively:

```text
vLLM
 │
 ├── Model: Qwen 2.5 1.5B Instruct AWQ
 ├── Quantization: AWQ
 ├── GPU Memory Utilization: 0.75
 ├── Max Model Length: 512
 └── Eager Execution
```

These settings are kept in the development overlay rather than hard-coded into the common base because runtime requirements can differ between environments.

---

# Development Secret Configuration

The file:

```text
secret-env-patch.yaml
```

connects the vLLM Deployment to a Kubernetes Secret.

The actual secret value is **not stored inside the Git repository**.

The architecture is:

```text
GitOps Repository
       │
       │ Secret Reference
       ▼
Kubernetes Secret
       │
       ▼
vLLM Deployment
       │
       ▼
     vLLM Pod
```

The development Secret is:

```text
vllm-api-secret
```

and exists in:

```text
vllm-dev
```

The GitOps repository only defines how the application references the Secret.

This prevents sensitive values from being committed directly into Git.

---

# Development Service

The development environment reuses the Service defined in the base.

```text
vLLM Pod
   │
   ▼
vLLM Service
   │
   │ Port 8000
   ▼
Internal Kubernetes Network
```

The Service selects the vLLM Pod using:

```text
app: vllm
```

The Service provides a stable endpoint even if the underlying Pod is recreated.

---

# Development Ingress

The development environment also reuses the Ingress from the base.

The traffic path is:

```text
Client
  │
  ▼
NGINX Ingress Controller
  │
  ▼
vLLM Ingress
  │
  ▼
vLLM Service :8000
  │
  ▼
vLLM Pod
```

The NGINX Ingress Controller was installed in the local Kubernetes cluster to provide the ingress layer.

---

# Development Monitoring

The development environment contains:

```text
vllm-servicemonitor.yaml
```

This defines a Prometheus `ServiceMonitor`.

The monitoring flow is:

```text
vLLM
 │
 │ /metrics
 ▼
ServiceMonitor
 │
 ▼
Prometheus
 │
 ▼
Grafana
```

The ServiceMonitor allows Prometheus to discover the vLLM metrics endpoint.

The monitoring configuration is associated with the vLLM Service through its labels.

---

# Development Alerting

The development environment also contains:

```text
monitoring/vllm-alerts.yaml
```

This defines a Prometheus alert rule.

The implemented alert is:

```text
VLLMPodDown
```

It checks whether the development vLLM Deployment has at least one available replica.

Conceptually:

```text
Available Replicas < 1
          │
          ▼
     VLLMPodDown
          │
          ▼
      Prometheus
          │
          ▼
      Alertmanager
          │
          ▼
        Slack
```

The alert was tested against the local development environment and successfully reached the configured Slack channel.

---

# Development GitOps Flow

The development environment is managed by Argo CD.

```text
GitHub
  │
  │ vllm-gitops
  ▼
overlays/dev/
  │
  ▼
Argo CD
  │
  │ Automated Sync
  ▼
Kubernetes
  │
  ▼
vllm-dev
  │
  ▼
vLLM Deployment
```

The Argo CD Application points to:

```text
Repository:
https://github.com/Sachin001234/vllm-gitops.git

Path:
overlays/dev

Namespace:
vllm-dev
```

Automated synchronization is enabled for the development application.

This allows Git changes to be automatically reconciled with the Kubernetes cluster.

---

# Development Self-Healing

Argo CD is configured with automated synchronization and self-healing for the development application.

The intended behavior is:

```text
Git Desired State
       │
       ▼
     Argo CD
       │
       ▼
 Kubernetes
       │
       │ Manual Drift
       ▼
Different State
       │
       ▼
 Argo CD Detects Drift
       │
       ▼
 Desired State Restored
```

This behavior was tested by manually changing the Deployment image.

Argo CD detected the drift and restored the Git-defined image.

---

# Development Rollback

The development environment was also used to test Git-based rollback.

The test flow was:

```text
Known-Good Commit
       │
       ▼
Controlled Bad Commit
       │
       ▼
Invalid Container Image
       │
       ▼
ImagePullBackOff
       │
       ▼
git revert
       │
       ▼
Known-Good Configuration
       │
       ▼
Argo CD Sync
```

This demonstrated that Git history can be used as the recovery mechanism for Kubernetes configuration.

---

# Local Kubernetes GPU Limitation

The development configuration is GPU-oriented because vLLM inference requires GPU resources for this project.

However, the local Docker Desktop Kubernetes cluster does not expose the NVIDIA GPU to Kubernetes.

The local GPU was successfully available to:

```text
Docker
   │
   ▼
vLLM Container
```

but not to:

```text
Docker Desktop Kubernetes
          │
          ▼
      vLLM Pod
          │
          ▼
     NVIDIA GPU
```

As a result, the development vLLM Deployment cannot become fully ready on the current local Kubernetes cluster.

This is why Argo CD may show the development application as:

```text
Synced
Progressing / Degraded
```

The GitOps configuration itself is synchronized correctly; the workload limitation comes from the local Kubernetes GPU environment.

---

# Development Environment Validation

The development environment can be inspected using:

```bash
kubectl get all -n vllm-dev
```

Check the Deployment:

```bash
kubectl get deployment -n vllm-dev
```

Check Pods:

```bash
kubectl get pods -n vllm-dev
```

Check the Service:

```bash
kubectl get svc -n vllm-dev
```

Check the Ingress:

```bash
kubectl get ingress -n vllm-dev
```

Check the ServiceMonitor:

```bash
kubectl get servicemonitor -n vllm-dev
```

Check the Prometheus alert rule:

```bash
kubectl get prometheusrule -n vllm-dev
```

Check the namespace:

```bash
kubectl get namespace vllm-dev
```

---

# Development Environment Summary

```text
                    GitHub
                       │
                       ▼
                vllm-gitops
                       │
                       ▼
                overlays/dev
                       │
              ┌────────┴────────┐
              ▼                 ▼
            base          Dev-specific
              │             patches
              │                 │
              └────────┬────────┘
                       ▼
                   Argo CD
                       │
                       ▼
                Kubernetes
                       │
                       ▼
                  vllm-dev
                       │
              ┌────────┼─────────┐
              ▼        ▼         ▼
           Ingress   Service   vLLM
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
                Prometheus               Alerts
                    │                       │
                    ▼                       ▼
                 Grafana                  Slack
```

The development overlay therefore provides a complete GitOps-managed environment for the vLLM platform, including **application deployment, networking, secrets references, monitoring, alerting, automated synchronization, self-healing, and rollback support**.

# 🚀 PART 6 — Production Environment

The `overlays/prod/` directory contains the **production-oriented Kubernetes configuration** for the vLLM platform.

The production overlay reuses the common resources from `base/` and adds configuration required for a GPU-enabled Kubernetes environment, resource management, controlled updates, topology awareness, secrets, and workload disruption protection.

> **Important:** The production configuration is prepared for a GPU-enabled Kubernetes cluster. It has not been deployed as a fully running production workload on the local Docker Desktop Kubernetes cluster because the local Kubernetes environment does not expose the NVIDIA GPU to Pods.

---

## Production Repository Structure

```text
overlays/prod/
│
├── kustomization.yaml
├── namespace.yaml
├── gpu-patch.yaml
├── replica-patch.yaml
├── resources-patch.yaml
├── rolling-update-patch.yaml
├── topology-patch.yaml
├── secret-env-patch.yaml
└── pdb.yaml
```

Each file has a specific responsibility.

```text
Production Overlay
        │
        ├── Namespace
        ├── GPU Scheduling
        ├── Replica Configuration
        ├── CPU / Memory Resources
        ├── Rolling Updates
        ├── Topology Configuration
        ├── Secret References
        └── PodDisruptionBudget
```

---

# Production Architecture

The production configuration follows this architecture:

```text
                    vllm-gitops
                         │
                         ▼
                   overlays/prod
                         │
                         ▼
                    Kustomize
                         │
                  ┌──────┴──────┐
                  ▼             ▼
                base        Prod Patches
                  │             │
                  │       ┌─────┼─────────────┐
                  │       ▼     ▼             ▼
                  │      GPU  Resources   Availability
                  │
                  └──────┬──────┘
                         ▼
                    Argo CD
                         │
                         ▼
              GPU-enabled Kubernetes
                         │
                         ▼
                   vLLM Deployment
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           Service    Ingress    Monitoring
```

---

# Production Namespace

The production environment uses a dedicated namespace:

```text
vllm-prod
```

This is defined in:

```text
overlays/prod/namespace.yaml
```

The namespace provides isolation between production and development workloads.

```text
Kubernetes Cluster
│
├── vllm-dev
│   └── Development vLLM
│
├── vllm-prod
│   └── Production vLLM
│
├── monitoring
│   └── Monitoring Stack
│
└── argocd
    └── Argo CD
```

---

# Production Kustomization

The production configuration is assembled through:

```text
overlays/prod/kustomization.yaml
```

The overlay starts with the common base:

```text
base/
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

and then applies production-specific patches.

```text
                     base
                      │
                      ▼
              Production Overlay
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
      GPU          Resources       Replicas
       │              │              │
       ├──────────────┼──────────────┤
       ▼              ▼              ▼
   Rolling Update   Topology         PDB
                      │
                      ▼
               Final Production
                  Manifests
```

This allows the production environment to reuse the same application architecture without duplicating the base manifests.

---

# GPU Configuration

The file:

```text
gpu-patch.yaml
```

adds GPU scheduling requirements to the vLLM Deployment.

The production workload requests:

```yaml
nvidia.com/gpu: 1
```

and uses an NVIDIA GPU node selector:

```text
accelerator: nvidia-gpu
```

The intended scheduling flow is:

```text
Kubernetes Scheduler
        │
        ▼
Node with:
accelerator=nvidia-gpu
        │
        ▼
NVIDIA GPU
        │
        ▼
vLLM Pod
```

This prevents Kubernetes from attempting to schedule the production vLLM workload onto a node that is not intended to provide the required GPU resource.

---

# GPU Resource Request

The production Deployment requests one NVIDIA GPU:

```text
nvidia.com/gpu: 1
```

This represents a real Kubernetes resource request.

Conceptually:

```text
vLLM Pod
   │
   └── GPU Requirement
          │
          └── 1 × NVIDIA GPU
```

A suitable Kubernetes cluster must therefore have:

- NVIDIA GPU hardware
- NVIDIA drivers
- NVIDIA container runtime support
- NVIDIA Kubernetes device plugin or equivalent GPU resource provider
- Nodes labeled appropriately

The local Docker Desktop Kubernetes environment used for this project does not currently expose the NVIDIA GPU as a Kubernetes resource.

---

# Production Resource Management

The file:

```text
resources-patch.yaml
```

defines CPU and memory requests and limits.

The current production configuration is:

```text
Requests:
  CPU:    2
  Memory: 8Gi

Limits:
  CPU:    4
  Memory: 12Gi
  GPU:    1
```

This gives Kubernetes explicit information about the resources required by the vLLM workload.

```text
              vLLM Pod
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
      CPU       Memory       GPU
       │          │          │
      2 CPU      8Gi         1 GPU
    request     request
       │          │
      4 CPU     12Gi
      limit      limit
```

Resource requests help the scheduler determine whether a node has enough available capacity.

Resource limits establish the maximum configured CPU and memory allocation for the container.

---

# Production Replica Configuration

The file:

```text
replica-patch.yaml
```

controls the number of vLLM replicas.

The current production configuration is:

```text
replicas: 1
```

This is an intentional project configuration.

It does **not** mean that the production environment currently provides high availability through multiple vLLM replicas.

Additional replicas would require additional suitable GPU capacity.

For example:

```text
Replica 1 → GPU Node 1
Replica 2 → GPU Node 2
Replica 3 → GPU Node 3
```

Each additional vLLM replica would require sufficient GPU resources to run the model.

---

# Rolling Update Strategy

The file:

```text
rolling-update-patch.yaml
```

configures the Deployment update strategy.

The production Deployment uses:

```text
strategy:
  type: RollingUpdate
```

with controlled update parameters.

The current configuration uses:

```text
maxSurge: 0
maxUnavailable: 1
```

This is particularly relevant for GPU workloads because creating an additional temporary vLLM Pod during an update could require another GPU.

The configured strategy avoids intentionally creating an additional surge replica during the update.

---

# Production Topology Configuration

The file:

```text
topology-patch.yaml
```

adds a topology spread constraint based on:

```text
kubernetes.io/hostname
```

The purpose is to provide Kubernetes with a distribution rule when multiple replicas are available.

Conceptually:

```text
Kubernetes Cluster

Node 1                  Node 2
┌─────────────┐         ┌─────────────┐
│ vLLM Pod 1  │         │ vLLM Pod 2  │
│    GPU      │         │    GPU      │
└─────────────┘         └─────────────┘
```

With the current configuration of only one replica, meaningful distribution cannot occur.

The topology configuration becomes more relevant when the deployment is scaled to multiple replicas and suitable GPU nodes are available.

---

# PodDisruptionBudget

The file:

```text
pdb.yaml
```

defines a Kubernetes `PodDisruptionBudget`.

The current configuration uses:

```text
minAvailable: 1
```

The PDB is intended to protect the workload from excessive voluntary disruption.

Conceptually:

```text
              Kubernetes
                  │
                  ▼
        Voluntary Disruption
                  │
                  ▼
          PodDisruptionBudget
                  │
                  ▼
       Keep required availability
```

With one configured replica, the PDB expresses that at least one vLLM Pod should remain available during voluntary disruptions.

Actual availability still depends on having a healthy, schedulable GPU node and a running Pod.

---

# Production Secrets

The file:

```text
secret-env-patch.yaml
```

contains references to Kubernetes Secrets required by the workload.

Secret values are **not stored in the GitOps repository**.

The architecture is:

```text
Git Repository
      │
      │ Secret Reference
      ▼
Kubernetes Secret
      │
      ▼
Production vLLM Pod
```

This follows the principle:

```text
Configuration → Git
Secret Values  → Kubernetes Secret Store
```

The GitOps repository therefore contains references to sensitive configuration without committing the actual secret values.

---

# Production Networking

The production overlay inherits the common networking resources from `base/`.

```text
Client
  │
  ▼
NGINX Ingress
  │
  ▼
vLLM Service
  │
  ▼
vLLM Pod
  │
  ▼
GPU
```

The production overlay does not need to duplicate the Service and Ingress manifests because those resources are already defined in the base.

This is one of the main advantages of the Kustomize base/overlay architecture.

---

# Production GitOps Flow

The production configuration is intended to be managed through Argo CD.

```text
GitHub
   │
   ▼
vllm-gitops
   │
   ▼
overlays/prod
   │
   ▼
Argo CD
   │
   ▼
GPU-enabled Kubernetes
   │
   ▼
vLLM Deployment
```

A production configuration change follows:

```text
Git Commit
    │
    ▼
Argo CD Detects Change
    │
    ▼
Kustomize Renders Manifests
    │
    ▼
Argo CD Synchronizes
    │
    ▼
Kubernetes Applies Change
```

This keeps production deployment changes traceable through Git history.

---

# Production Readiness Features

The production overlay contains several production-oriented capabilities:

```text
Production Configuration
        │
        ├── GPU Scheduling
        ├── Resource Requests/Limits
        ├── Replica Configuration
        ├── Rolling Updates
        ├── Topology Awareness
        ├── PodDisruptionBudget
        ├── Secret References
        └── Namespace Isolation
```

These features provide the Kubernetes configuration needed to move the workload toward a production GPU environment.

They do not by themselves guarantee production availability; actual production behavior also depends on the underlying Kubernetes infrastructure, GPU capacity, networking, storage, observability, security, and operational practices.

---

# Local Environment Limitation

The production overlay is **GPU-ready but not locally runnable on the current Docker Desktop Kubernetes cluster**.

The local environment has:

```text
Windows
   │
   ▼
WSL2
   │
   ▼
Docker Desktop
   │
   ├── Docker GPU support → ✅
   │
   └── Kubernetes GPU resource → ❌
```

The RTX 3050 Ti Laptop GPU was successfully exposed to Docker containers, allowing local vLLM inference.

However, the same GPU was not exposed as:

```text
nvidia.com/gpu
```

inside Docker Desktop Kubernetes.

Therefore:

```text
Docker + vLLM
      │
      └── GPU inference works

Docker Desktop Kubernetes + vLLM
      │
      └── GPU scheduling unavailable locally
```

This limitation is documented rather than hiding the actual deployment state.

---

# Production Validation

The production manifests can be rendered without deploying them using:

```bash
kubectl kustomize overlays/prod
```

This allows the generated Kubernetes configuration to be inspected before deployment.

The rendered configuration can be saved for inspection:

```bash
kubectl kustomize overlays/prod > /tmp/vllm-prod.yaml
```

The production Deployment can then be inspected:

```bash
kubectl kustomize overlays/prod | grep -A20 "kind: Deployment"
```

GPU configuration can be checked with:

```bash
kubectl kustomize overlays/prod | grep -A10 "nvidia.com/gpu"
```

The namespace can be checked with:

```bash
kubectl kustomize overlays/prod | grep "name: vllm-prod"
```

---

# Production Architecture Summary

```text
                         GitHub
                            │
                            ▼
                     vllm-gitops
                            │
                            ▼
                     overlays/prod
                            │
                            ▼
                         Kustomize
                            │
               ┌────────────┼────────────┐
               ▼            ▼            ▼
              Base        GPU         Resources
               │          Config        │
               │            │            │
               └────────────┼────────────┘
                            ▼
                         Argo CD
                            │
                            ▼
                 GPU-enabled Kubernetes
                            │
                     ┌──────┴──────┐
                     ▼             ▼
                vllm-prod      GPU Node
                     │             │
                     └──────┬──────┘
                            ▼
                         vLLM Pod
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
             Service     Ingress      GPU
                │
                ▼
             vLLM API
```

The production overlay therefore provides a **GPU-aware, resource-controlled, GitOps-managed Kubernetes configuration** for the vLLM platform, while clearly separating production-oriented configuration from the development environment.

# 📊 Monitoring & Alerting

The `vllm-gitops` repository manages the Kubernetes monitoring and alerting configuration required to observe the vLLM platform.

The monitoring architecture uses:

- Prometheus
- Grafana
- Prometheus Operator
- ServiceMonitor
- PrometheusRule
- Alertmanager
- Slack

The overall flow is:

```text
                         vLLM
                           │
                           │ /metrics
                           ▼
                    ┌───────────────┐
                    │ ServiceMonitor│
                    └───────┬───────┘
                            │
                            ▼
                     ┌─────────────┐
                     │  Prometheus │
                     └──────┬──────┘
                            │
                ┌───────────┴───────────┐
                │                       │
                ▼                       ▼
             Grafana              Alert Rules
                │                       │
                │                       ▼
                │                 Alertmanager
                │                       │
                │                       ▼
                │                     Slack
                │
                ▼
             Dashboards
```

---

## Monitoring Stack

The Kubernetes monitoring stack was installed using the `kube-prometheus-stack`.

The stack provides:

```text
Prometheus
    │
    ├── Metrics collection
    │
    ├── Alert evaluation
    │
    ▼
Grafana
    │
    └── Visualization

Alertmanager
    │
    └── Notification routing
```

The monitoring components run in the:

```text
monitoring
```

namespace.

---

## Prometheus

Prometheus is responsible for collecting and storing metrics from the Kubernetes environment and vLLM application.

The project uses Prometheus to monitor:

- vLLM metrics
- Kubernetes workload metrics
- Pod status
- Deployment availability
- Resource utilization
- GPU-related metrics when available
- Application traffic and token metrics

The Prometheus service is exposed internally on:

```text
9090
```

Prometheus readiness can be checked with:

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Then:

```bash
curl -s http://localhost:9090/-/ready
```

A successful response confirms that Prometheus is ready.

---

## vLLM Metrics Collection

The vLLM server exposes Prometheus-compatible metrics through:

```text
/metrics
```

The monitoring architecture is:

```text
vLLM
 │
 └── /metrics
       │
       ▼
ServiceMonitor
       │
       ▼
Prometheus
```

This allows application-level inference metrics to be collected alongside Kubernetes infrastructure metrics.

---

## ServiceMonitor

The development environment contains:

```text
overlays/dev/vllm-servicemonitor.yaml
```

This defines a Prometheus `ServiceMonitor`.

Its purpose is to tell Prometheus:

```text
Which Service?
        │
        ▼
vLLM Service

Which endpoint?
        │
        ▼
/metrics

How frequently?
        │
        ▼
15 seconds
```

The configured monitoring endpoint uses:

```text
Path:
 /metrics

Interval:
 15s
```

The ServiceMonitor selects the vLLM Service using its Kubernetes labels.

The resource can be checked with:

```bash
kubectl get servicemonitor -n vllm-dev
```

---

## Grafana

Grafana is used to visualize the collected metrics.

The project contains a dashboard named:

```text
vLLM Production Monitoring
```

The dashboard provides visibility into both vLLM and Kubernetes metrics.

The monitoring flow is:

```text
vLLM / Kubernetes
        │
        ▼
    Prometheus
        │
        ▼
     Grafana
        │
        ▼
    Dashboards
```

---

## vLLM Metrics

The dashboard includes vLLM-related metrics such as:

```text
vllm:num_requests_running
```

This provides visibility into the number of currently running requests.

```text
vllm:kv_cache_usage_perc
```

This provides visibility into KV cache utilization.

Token-related metrics include:

```text
vllm:prompt_tokens_total
vllm:generation_tokens_total
```

These provide visibility into prompt and generated token activity.

---

## Kubernetes Metrics

The Grafana dashboard also includes Kubernetes workload metrics.

Examples include:

```text
kube_deployment_status_replicas
```

for Deployment replica state.

Pod restart metrics are used to identify workload instability.

Pod phase metrics provide visibility into states such as:

```text
Running
Pending
Failed
Succeeded
Unknown
```

CPU and memory metrics are also included for workload resource visibility.

---

## GPU Monitoring

The monitoring dashboard also includes a GPU availability metric:

```text
count(DCGM_FI_DEV_GPU_UTIL)
```

GPU metrics are particularly important for the vLLM platform because inference workloads depend on GPU resources.

The expected architecture in a GPU-enabled environment is:

```text
NVIDIA GPU
     │
     ▼
DCGM Exporter
     │
     ▼
Prometheus
     │
     ▼
Grafana
```

The local Docker environment can access the NVIDIA GPU, but the local Docker Desktop Kubernetes environment does not expose the GPU to Kubernetes Pods.

Therefore GPU-related Kubernetes metrics are limited in the local cluster.

---

## Monitoring GitOps Configuration

Monitoring configuration is stored in Git rather than being maintained manually inside the cluster.

The repository contains:

```text
monitoring/
├── kustomization.yaml
└── vllm-slack.yaml
```

The development overlay contains:

```text
overlays/dev/
├── vllm-servicemonitor.yaml
└── monitoring/
    └── vllm-alerts.yaml
```

This separates monitoring responsibilities:

```text
monitoring/
    │
    └── Alertmanager / notification configuration

overlays/dev/
    │
    ├── ServiceMonitor
    │
    └── Prometheus alert rules
```

---

# Alerting

The project uses Prometheus alert rules to detect important workload conditions.

The alerting architecture is:

```text
                 Kubernetes
                      │
                      ▼
                  Prometheus
                      │
                 Alert Rules
                      │
                      ▼
                 Alertmanager
                      │
                      ▼
                    Slack
```

---

## PrometheusRule

The development alert configuration is located at:

```text
overlays/dev/monitoring/vllm-alerts.yaml
```

It defines the custom alert:

```text
VLLMPodDown
```

The alert checks whether the development vLLM Deployment has an available replica.

Conceptually:

```text
Available vLLM Replicas
          │
          ▼
       < 1 ?
          │
      ┌───┴───┐
      │       │
     Yes      No
      │       │
      ▼       ▼
    Alert    Normal
```

The alert is configured with:

```text
Severity:
critical

Duration:
2 minutes
```

This means the condition must persist for the configured duration before the alert fires.

---

## Alert Rule

The implemented rule checks:

```text
kube_deployment_status_replicas_available
```

for:

```text
namespace = vllm-dev
deployment = dev-vllm
```

and triggers when:

```text
available replicas < 1
```

This prevents a temporary state from immediately generating an alert.

---

## Alertmanager

Prometheus evaluates the alert rule, while **Alertmanager handles notification routing**.

The flow is:

```text
Prometheus
    │
    │ VLLMPodDown fires
    ▼
Alertmanager
    │
    │ Match app=vllm
    ▼
Slack Receiver
    │
    ▼
#vllm-alerts
```

Alertmanager is configured through:

```text
monitoring/vllm-slack.yaml
```

---

## Slack Integration

Slack was selected as the notification destination for vLLM alerts.

The notification flow is:

```text
Prometheus
    │
    ▼
Alertmanager
    │
    ▼
Slack Webhook
    │
    ▼
#vllm-alerts
```

The Slack webhook URL is **not stored directly in Git**.

Instead, Alertmanager references a Kubernetes Secret:

```text
slack-webhook
```

The Secret contains the webhook value, while Git only contains the reference.

```text
GitOps
  │
  └── Secret Reference
          │
          ▼
   Kubernetes Secret
          │
          ▼
     Alertmanager
          │
          ▼
        Slack
```

---

## AlertmanagerConfig

The file:

```text
monitoring/vllm-slack.yaml
```

defines the Alertmanager configuration.

The configuration includes:

```text
Receiver:
vllm-slack

Channel:
#vllm-alerts

Grouping:
alertname

Group Wait:
10 seconds

Group Interval:
5 minutes

Repeat Interval:
4 hours
```

Resolved notifications are also configured using:

```text
sendResolved: true
```

This allows Alertmanager to notify Slack when a previously firing alert returns to a resolved state.

---

## Cross-Namespace Alert Routing

The vLLM alert exists in:

```text
vllm-dev
```

while the Alertmanager configuration exists in:

```text
monitoring
```

The project therefore configures Alertmanager to allow AlertmanagerConfig resources to be processed across namespaces.

The configured matcher strategy is:

```text
None
```

This allows the monitoring configuration in the `monitoring` namespace to process the vLLM alert generated in the `vllm-dev` namespace.

The resulting flow is:

```text
vllm-dev
   │
   │ PrometheusRule
   ▼
Prometheus
   │
   ▼
Alertmanager
   │
   │ monitoring/vllm-slack
   ▼
Slack
```

---

## Alert Testing

The alerting system was tested using the actual development workload state.

Because the local Kubernetes environment cannot provide the required GPU to the vLLM Pod, the development Deployment had no available replica.

This caused:

```text
VLLMPodDown
```

to enter the firing state.

The alert was successfully delivered to:

```text
#vllm-alerts
```

The Slack channel showed the firing notification:

```text
[FIRING:1] VLLMPodDown
```

This verified the complete notification path:

```text
Kubernetes
    │
    ▼
Prometheus
    │
    ▼
VLLMPodDown
    │
    ▼
Alertmanager
    │
    ▼
Slack
```

---

## Alerting Security

Sensitive notification credentials are kept outside Git.

The repository contains only the reference:

```text
slack-webhook
```

The actual webhook value is stored as a Kubernetes Secret.

```text
❌ Git Repository
   └── Actual webhook URL

✅ Git Repository
   └── Secret reference

✅ Kubernetes
   └── Actual secret value
```

This prevents the webhook credential from being committed to the Git repository.

---

## GitOps Monitoring Flow

Monitoring configuration is also reconciled through GitOps.

```text
Developer
    │
    ▼
vllm-gitops
    │
    ├── ServiceMonitor
    ├── PrometheusRule
    └── AlertmanagerConfig
    │
    ▼
Argo CD
    │
    ▼
Kubernetes
    │
    ├── Prometheus
    ├── Grafana
    └── Alertmanager
             │
             ▼
           Slack
```

Changes to monitoring configuration can therefore be reviewed and tracked through Git history.

---

## Monitoring Validation Commands

Check the monitoring namespace:

```bash
kubectl get pods -n monitoring
```

Check Prometheus:

```bash
kubectl get prometheus -n monitoring
```

Check Grafana:

```bash
kubectl get svc -n monitoring | grep grafana
```

Check Alertmanager:

```bash
kubectl get alertmanager -n monitoring
```

Check the vLLM ServiceMonitor:

```bash
kubectl get servicemonitor -n vllm-dev
```

Check the PrometheusRule:

```bash
kubectl get prometheusrule -n vllm-dev
```

Check the AlertmanagerConfig:

```bash
kubectl get alertmanagerconfig -n monitoring
```

Check the Slack Secret:

```bash
kubectl get secret slack-webhook -n monitoring
```

The Secret should be inspected only as metadata. The actual webhook value should never be printed unnecessarily.

---

## Monitoring Limitation in Local Kubernetes

The monitoring stack itself is functioning in the local Kubernetes environment.

However, vLLM application metrics are limited because the vLLM Pod cannot successfully start without a Kubernetes-visible GPU.

Therefore:

```text
Prometheus
   │
   ├── Kubernetes metrics → Available
   │
   ├── Deployment metrics → Available
   │
   └── vLLM runtime metrics → Limited
                            │
                            └── Pod requires GPU
```

This is an infrastructure limitation rather than a monitoring configuration failure.

The same GitOps monitoring configuration can be used in a GPU-enabled Kubernetes environment where the vLLM workload is able to run.

---

## Complete Monitoring & Alerting Architecture

```text
                           vLLM
                            │
                            │ /metrics
                            ▼
                     ┌──────────────┐
                     │ ServiceMonitor│
                     └──────┬───────┘
                            │
                            ▼
                     ┌─────────────┐
                     │ Prometheus  │
                     └──────┬──────┘
                            │
                ┌───────────┴────────────┐
                │                        │
                ▼                        ▼
             Grafana               PrometheusRule
                │                        │
                ▼                        ▼
           Dashboards              VLLMPodDown
                                         │
                                         ▼
                                  Alertmanager
                                         │
                                         ▼
                                  Slack Webhook
                                         │
                                         ▼
                                  #vllm-alerts
```

The monitoring architecture provides the vLLM platform with **metrics collection, visualization, workload alerting, notification routing, and GitOps-managed observability configuration**.

# 🔐 Secrets Management

The `vllm-gitops` repository follows a security-focused approach for handling sensitive configuration.

Secret values are **not stored directly in Git**. Instead, the GitOps repository stores references to Kubernetes Secrets, while the actual sensitive values are created and maintained inside the Kubernetes cluster.

The basic principle is:

```text
Git Repository
      │
      │ Secret Reference
      ▼
Kubernetes Secret
      │
      ▼
Application / Monitoring Component
```

This approach is used for:

- vLLM API credentials
- Slack webhook credentials
- GitHub Actions authentication
- Other sensitive configuration that may be introduced later

---

## Secret Management Architecture

The project separates normal configuration from sensitive values.

```text
                 GitOps Repository
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
   Normal Configuration        Secret References
          │                           │
          │                           ▼
          │                   Kubernetes Secrets
          │                           │
          └─────────────┬─────────────┘
                        ▼
                    Kubernetes
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
            vLLM              Alertmanager
```

The repository can therefore describe **which Secret should be used** without containing the Secret's actual value.

---

# Kubernetes Secrets

Kubernetes Secrets are used to store sensitive configuration inside the cluster.

Examples in this project include:

```text
vllm-api-secret
slack-webhook
```

These Secrets are associated with their respective namespaces.

```text
vllm-dev
   │
   └── vllm-api-secret

monitoring
   │
   └── slack-webhook
```

Kubernetes namespaces provide separation between these Secret objects.

---

# vLLM API Secret

The development vLLM deployment references:

```text
vllm-api-secret
```

This Secret exists in:

```text
vllm-dev
```

The GitOps repository does not contain the actual credential value.

Instead, the development overlay contains the configuration required to reference the Secret.

```text
overlays/dev/
        │
        └── secret-env-patch.yaml
                    │
                    ▼
             vllm-api-secret
                    │
                    ▼
                vLLM Pod
```

This keeps the credential separate from the Kubernetes manifests stored in Git.

---

# Secret Reference Pattern

The general pattern used by the project is:

```text
Git
 │
 └── Reference to Secret
          │
          ▼
 Kubernetes
 │
 └── Secret object
          │
          ▼
       Workload
```

The GitOps repository therefore contains declarative configuration such as:

```text
Use Secret: vllm-api-secret
```

rather than:

```text
API_KEY=actual-secret-value
```

---

# Slack Webhook Secret

The monitoring system uses a Kubernetes Secret named:

```text
slack-webhook
```

The Secret exists in:

```text
monitoring
```

namespace.

It is used by the Alertmanager Slack configuration.

```text
AlertmanagerConfig
       │
       ▼
slack-webhook Secret
       │
       ▼
Webhook Credential
       │
       ▼
Slack
```

The actual Slack webhook URL is **not stored in the Git repository**.

The Alertmanager configuration only references the Secret and its key.

---

# Alertmanager Secret Reference

The GitOps monitoring configuration uses the following conceptual structure:

```yaml
apiURL:
  name: slack-webhook
  key: webhook-url
```

This means:

```text
Alertmanager
     │
     ▼
Secret:
slack-webhook
     │
     ▼
Key:
webhook-url
     │
     ▼
Actual webhook value
```

The sensitive webhook value remains outside the Git repository.

---

# GitHub Actions Secret

The application repository also requires a credential to allow GitHub Actions to update the GitOps repository.

The credential is stored as a GitHub Actions repository Secret:

```text
VLLM_GITOPS_TOKEN
```

The architecture is:

```text
GitHub Actions
      │
      │ VLLM_GITOPS_TOKEN
      ▼
GitHub API
      │
      ▼
vllm-gitops
```

The token allows the CI pipeline in `vllm-platform` to update the image reference in `vllm-gitops`.

The token itself is not committed to either repository.

---

# Cross-Repository Authentication

The two-repository architecture requires controlled authentication between:

```text
vllm-platform
        │
        │ GitHub Actions
        ▼
vllm-gitops
```

The CI workflow uses:

```text
VLLM_GITOPS_TOKEN
```

to authenticate when cloning and pushing changes to the GitOps repository.

The flow is:

```text
Code Push
    │
    ▼
GitHub Actions
    │
    ├── Test
    ├── Build
    ├── Push Image
    │
    ▼
Authenticate to GitOps Repository
    │
    ▼
Update Kubernetes Image Reference
    │
    ▼
GitOps Commit
```

This allows application CI and Kubernetes deployment configuration to remain in separate repositories.

---

# Secret Creation

Secrets are created outside the Git-tracked Kubernetes manifests.

For example, a development Secret can be created using:

```bash
kubectl create secret generic vllm-api-secret \
  -n vllm-dev \
  --from-literal=api-key='<secret-value>'
```

The actual secret value should be supplied securely and should not be committed to Git.

Similarly, the Slack webhook Secret can be created separately:

```bash
kubectl create secret generic slack-webhook \
  -n monitoring \
  --from-literal=webhook-url='<webhook-value>'
```

The webhook value should never be placed directly inside:

```text
vllm-slack.yaml
```

---

# Secret Verification

Secret existence can be checked without exposing the value.

For the vLLM Secret:

```bash
kubectl get secret vllm-api-secret -n vllm-dev
```

For the Slack Secret:

```bash
kubectl get secret slack-webhook -n monitoring
```

To inspect Secret metadata:

```bash
kubectl describe secret vllm-api-secret -n vllm-dev
```

and:

```bash
kubectl describe secret slack-webhook -n monitoring
```

The actual Secret values should not be printed unnecessarily.

---

# Secrets and Kustomize

Kustomize overlays contain Secret references rather than sensitive values.

The development structure is:

```text
overlays/dev/
│
├── kustomization.yaml
├── secret-env-patch.yaml
└── ...
```

The production structure also contains:

```text
overlays/prod/
│
├── kustomization.yaml
├── secret-env-patch.yaml
└── ...
```

This allows each environment to reference its own Kubernetes Secret.

```text
             Kustomize
                 │
       ┌─────────┴─────────┐
       ▼                   ▼
      Dev                 Prod
       │                   │
       ▼                   ▼
vllm-api-secret       Production Secret
```

The Secret values themselves remain outside the Git repository.

---

# Namespace Isolation

Secrets are namespace-scoped Kubernetes resources.

The project therefore separates them according to workload:

```text
vllm-dev
   │
   └── vllm-api-secret

monitoring
   │
   └── slack-webhook

vllm-prod
   │
   └── Production Secret
```

This prevents a Secret from automatically being available across namespaces.

A workload must reference a Secret in the appropriate namespace.

---

# GitOps and Secret Security

GitOps provides strong configuration traceability, but sensitive values should not simply be committed to Git.

The project follows:

```text
                    Git
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
Non-sensitive config        Secret references
        │                         │
        │                         ▼
        │                  Kubernetes Secret
        │                         │
        └────────────┬────────────┘
                     ▼
                 Kubernetes
```

This means Git can safely track:

- Secret names
- Secret references
- Environment configuration
- Application configuration

while sensitive values are maintained separately.

---

# What Is Not Stored in Git

The following sensitive values are intentionally excluded from the repository:

```text
❌ vLLM API secret values
❌ Slack webhook URL
❌ GitHub access token
❌ Other authentication credentials
```

The repository contains only the configuration required to reference these credentials.

---

# Security Principle

The project follows the principle:

```text
Configuration belongs in Git.
Secrets belong outside Git.
```

More specifically:

```text
┌─────────────────────────────┐
│            Git              │
│                             │
│ Kubernetes manifests        │
│ Kustomize configuration     │
│ Secret references           │
│ Monitoring configuration    │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│        Kubernetes           │
│                             │
│ Secret values               │
│ Runtime credentials         │
│ Webhook credentials         │
└─────────────────────────────┘
```

---

# Secret Management Flow

The complete secret flow is:

```text
                    Developer / Operator
                            │
                            │ Secure Secret Creation
                            ▼
                    Kubernetes Secret
                            │
             ┌──────────────┴──────────────┐
             ▼                             ▼
        vLLM Workload                Alertmanager
             │                             │
             │                             │
             ▼                             ▼
       Application                   Slack Webhook
       Credentials                  Authentication
```

For CI/CD:

```text
GitHub Actions
      │
      ▼
VLLM_GITOPS_TOKEN
      │
      ▼
GitHub Repository
      │
      ▼
vllm-gitops
```

The project therefore keeps **application credentials, notification credentials, and CI authentication credentials outside the Git-tracked configuration**, while still allowing the GitOps system to reference the required Kubernetes Secrets declaratively.

# 🔄 Rollback & Self-Healing

The vLLM GitOps platform uses **Git-based rollback, Argo CD self-healing, and Kubernetes workload recovery** to recover from deployment failures and configuration drift.

The recovery model is:

```text
                    Git Repository
                          │
                          ▼
                       Argo CD
                          │
                          ▼
                     Kubernetes
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
        Configuration             Workload
           Drift                   Failure
              │                       │
              ▼                       ▼
        Argo CD Self-Heal       Kubernetes Recovery
              │                       │
              └───────────┬───────────┘
                          ▼
                    Desired State
```

Git history provides the primary rollback mechanism, while Argo CD and Kubernetes provide automated reconciliation and workload recovery.

---

## Rollback Strategy

The project uses Git as the source of truth.

Instead of manually editing the Kubernetes Deployment during a failed deployment, the desired configuration is corrected in Git.

```text
Git
 │
 ├── Known-Good Commit
 │
 ├── New Deployment Commit
 │
 └── Rollback Commit
          │
          ▼
       Argo CD
          │
          ▼
     Kubernetes
```

This provides:

- Version-controlled deployment history
- Traceable configuration changes
- Reproducible deployments
- Git-based rollback
- Argo CD reconciliation after rollback

---

# Controlled Deployment Failure

A controlled rollback test was performed using an intentionally invalid container image.

A temporary patch changed the vLLM image to:

```text
ghcr.io/sachin001234/vllm-platform:rollback-test-invalid
```

The change was added through Git rather than manually changing the Kubernetes Deployment.

The test flow was:

```text
Known-Good Image
       │
       ▼
GitOps Change
       │
       ▼
Invalid Image
       │
       ▼
Argo CD Sync
       │
       ▼
Kubernetes Deployment
       │
       ▼
ImagePullBackOff
```

The resulting failure was expected because the specified image did not exist.

This provided a controlled way to verify the rollback process.

---

# Git-Based Rollback

After the invalid deployment was confirmed, the Git commit was reverted.

The rollback flow was:

```text
Bad Commit
    │
    ▼
git revert
    │
    ▼
Revert Commit
    │
    ▼
GitHub
    │
    ▼
Argo CD
    │
    ▼
Kubernetes
    │
    ▼
Known-Good Image
```

The rollback was performed using:

```bash
git revert <bad-commit>
```

The revert created a new Git commit rather than rewriting the repository history.

This preserves the complete sequence of events:

```text
Known-Good
    │
    ▼
Bad Deployment
    │
    ▼
Rollback
```

---

# Argo CD Reconciliation After Rollback

After the Git revert was pushed, Argo CD detected the new Git revision.

```text
Git Revert
    │
    ▼
Argo CD detects revision
    │
    ▼
Desired state changes
    │
    ▼
Argo CD synchronization
    │
    ▼
Kubernetes Deployment
    │
    ▼
Known-Good image restored
```

The Argo CD application returned to the Git-defined image reference.

The known-good image used during the project was:

```text
ghcr.io/sachin001234/vllm-platform:b2fdadc53eab4c42e7ca657400a114f8a2311722
```

Using the commit SHA as the image tag makes it possible to associate the deployed image with a specific source revision.

---

# Self-Healing

Argo CD self-healing protects the Kubernetes environment from configuration drift.

Configuration drift occurs when the actual Kubernetes state differs from the state stored in Git.

```text
Git Desired State
       │
       ▼
     Argo CD
       │
       ▼
 Kubernetes
       │
       │
       ▼
 Manual Change
       │
       ▼
    Drift
       │
       ▼
 Argo CD detects difference
       │
       ▼
 Desired state restored
```

The development Argo CD Application has automated synchronization and self-healing enabled.

---

# Self-Healing Test

Self-healing was tested by manually changing the Kubernetes Deployment image.

The image was temporarily changed to:

```text
manual-drift-test
```

This created a deliberate difference between:

```text
Git:
Known-good image

Kubernetes:
manual-drift-test
```

Argo CD detected the difference.

```text
Git
 │
 └── Known-Good Image
          │
          ▼
      Argo CD
          │
          │ Detect Drift
          ▼
    Kubernetes
          │
          └── manual-drift-test
                  │
                  ▼
             Reconciliation
                  │
                  ▼
             Known-Good Image
```

The Deployment was subsequently restored to the image defined by Git.

This demonstrated Argo CD self-healing behavior.

---

# Kubernetes Pod Recovery

Kubernetes provides another level of automatic recovery.

The vLLM workload is managed by a Deployment.

```text
Deployment
    │
    ▼
ReplicaSet
    │
    ▼
Pod
```

When the Pod was manually deleted:

```bash
kubectl delete pod <vllm-pod> -n vllm-dev
```

the ReplicaSet detected that the desired number of Pods was no longer available and created a replacement.

```text
Original Pod
     │
     │ Deleted
     ▼
ReplicaSet
     │
     ▼
Replacement Pod
```

This demonstrates Kubernetes workload reconciliation independently of Argo CD.

---

# Difference Between Argo CD and Kubernetes Recovery

The project uses two different recovery mechanisms.

### Argo CD

Responsible for **configuration reconciliation**.

```text
Git
 │
 ▼
Argo CD
 │
 ▼
Kubernetes Configuration
```

It restores resources when the cluster configuration differs from the Git desired state.

### Kubernetes

Responsible for **workload reconciliation**.

```text
Deployment
 │
 ▼
ReplicaSet
 │
 ▼
Pod
```

It recreates Pods when the desired number of replicas is not running.

Together:

```text
Git
 │
 ▼
Argo CD
 │
 ▼
Deployment
 │
 ▼
ReplicaSet
 │
 ▼
Pod
```

Each layer has a different recovery responsibility.

---

# Rollback vs Self-Healing

These mechanisms solve different problems.

| Mechanism | Purpose |
|---|---|
| Git Revert | Roll back an intentionally changed deployment configuration |
| Argo CD Sync | Apply the desired Git state to Kubernetes |
| Argo CD Self-Heal | Correct configuration drift |
| Kubernetes Deployment | Maintain the desired replica count |
| ReplicaSet | Recreate missing Pods |
| Pod Restart | Recover from certain container failures |

The overall recovery model is:

```text
Deployment Configuration Failure
            │
            ▼
        Git Revert
            │
            ▼
         Argo CD
            │
            ▼
       Correct State
```

and:

```text
Pod Failure
    │
    ▼
ReplicaSet
    │
    ▼
Replacement Pod
```

---

# Rollback Test Results

The project successfully demonstrated the following:

```text
✔ Controlled invalid image deployed
✔ Kubernetes detected invalid image
✔ Pod entered ImagePullBackOff
✔ Bad Git commit was reverted
✔ Argo CD detected the revert
✔ Known-good image was restored
✔ Manual configuration drift was introduced
✔ Argo CD self-healed the drift
✔ Pod deletion triggered Kubernetes replacement
```

The local vLLM Pod could not become fully healthy after recreation because the Docker Desktop Kubernetes environment does not expose the NVIDIA GPU required by vLLM.

The recovery mechanisms themselves were still verified.

---

# GitOps Recovery Architecture

```text
                     GitHub
                        │
                        ▼
                  vllm-gitops
                        │
                        ▼
                     Argo CD
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
          Sync / Self-Heal     Rollback
              │                   │
              └─────────┬─────────┘
                        ▼
                   Kubernetes
                        │
                        ▼
                   Deployment
                        │
                        ▼
                    ReplicaSet
                        │
                        ▼
                      Pod
```

This architecture provides recovery at multiple layers:

```text
Git Layer
    │
    └── Version history / rollback

Argo CD Layer
    │
    └── Synchronization / self-healing

Kubernetes Layer
    │
    └── Pod recreation / workload recovery
```

---

# Validation Commands

Check the Argo CD Application:

```bash
kubectl get application vllm-dev -n argocd
```

Inspect synchronization and health:

```bash
kubectl describe application vllm-dev -n argocd
```

Check the Deployment:

```bash
kubectl get deployment -n vllm-dev
```

Check ReplicaSets:

```bash
kubectl get rs -n vllm-dev
```

Check Pods:

```bash
kubectl get pods -n vllm-dev
```

Check the deployed image:

```bash
kubectl get deployment dev-vllm \
  -n vllm-dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Check Git history:

```bash
git log --oneline --decorate -10
```

---

# Recovery Model

The complete recovery process can be summarized as:

```text
                         Git
                          │
                 ┌────────┴────────┐
                 │                 │
             Bad Change        Desired State
                 │                 │
                 ▼                 ▼
            Git Revert         Argo CD
                 │                 │
                 └────────┬────────┘
                          ▼
                     Kubernetes
                          │
                    ┌─────┴─────┐
                    ▼           ▼
              Config Drift   Pod Failure
                    │           │
                    ▼           ▼
                Argo CD      ReplicaSet
                Self-Heal        │
                    │            ▼
                    │       New Pod
                    │
                    └─────┬─────┘
                          ▼
                    Recovered State
```

The project therefore combines **Git-based rollback, Argo CD self-healing, and Kubernetes workload recovery** to create a resilient GitOps deployment model for the vLLM platform.

# 🚀 Deployment Flow

The vLLM platform uses a **CI/CD + GitOps deployment flow** in which application changes are built and tested in the `vllm-platform` repository, while Kubernetes deployment state is maintained separately in the `vllm-gitops` repository.

The complete deployment pipeline is:

```text
Developer
    │
    │ Push code
    ▼
vllm-platform
    │
    ▼
GitHub Actions
    │
    ├── Run tests
    ├── Build Docker image
    └── Push image to GHCR
    │
    ▼
Update vllm-gitops
    │
    │ Update image tag
    ▼
vllm-gitops
    │
    ▼
Argo CD
    │
    ├── Detect Git change
    ├── Render Kustomize
    └── Synchronize
    │
    ▼
Kubernetes
    │
    ▼
vLLM Deployment
    │
    ├── Service
    └── Ingress
    │
    ▼
vLLM API
```

---

## Application Change

The deployment process begins when a developer makes a change to the vLLM platform.

The application-side repository is:

```text
vllm-platform
```

Typical changes may include:

- vLLM configuration
- Docker configuration
- Test changes
- Platform scripts
- Runtime configuration
- CI/CD configuration

The developer commits and pushes the changes to GitHub.

```text
Developer
    │
    ▼
Git commit
    │
    ▼
Git push
    │
    ▼
vllm-platform
```

---

# GitHub Actions CI

A push to the `main` branch triggers the GitHub Actions workflow.

```text
GitHub Push
     │
     ▼
GitHub Actions
     │
     ▼
CI Workflow
```

The workflow performs automated validation before publishing a new container image.

The main stages are:

```text
Checkout
   │
   ▼
Install Dependencies
   │
   ▼
Run Tests
   │
   ▼
Build Docker Image
   │
   ▼
Push Image to GHCR
```

If the tests fail, the workflow does not proceed with a successful deployment image.

---

# Automated Testing

The CI pipeline runs the project's CI-safe tests.

```bash
pytest -v -m "not integration"
```

The CI environment does not start a GPU-based vLLM server, so the full local integration tests are excluded from the CI-safe stage.

The pipeline therefore separates:

```text
CI Environment
    │
    └── Configuration / unit-style tests

Local GPU Environment
    │
    └── vLLM integration tests
```

The project previously verified:

```text
Local integration tests:
3 passed

CI-safe tests:
1 passed, 3 deselected
```

---

# Docker Image Build

After successful testing, GitHub Actions builds the vLLM platform container image.

The image is based on the official vLLM container image:

```text
vllm/vllm-openai:v0.30.0
```

The resulting project image is published to GitHub Container Registry.

```text
GitHub Actions
      │
      ▼
Docker Build
      │
      ▼
vllm-platform Image
      │
      ▼
GHCR
```

---

# Immutable Image Tagging

The container image is tagged using the Git commit SHA.

Example:

```text
ghcr.io/sachin001234/vllm-platform:<commit-sha>
```

This provides a traceable relationship between the source code and the container image.

```text
Git Commit
     │
     ▼
Commit SHA
     │
     ├── Docker Image Tag
     │
     └── GitOps Image Reference
```

For example:

```text
Git Commit
     │
     ▼
b2fdadc...
     │
     ▼
ghcr.io/sachin001234/vllm-platform:b2fdadc...
```

Using immutable image references also makes rollback easier because a previous image can be referenced directly.

---

# GitOps Repository Update

After the container image is successfully pushed, GitHub Actions updates the Kubernetes image reference in:

```text
vllm-gitops
```

The image reference in the GitOps Deployment is changed from the previous image SHA to the new one.

Conceptually:

```text
Before:

image: ghcr.io/sachin001234/vllm-platform:OLD_SHA
```

After:

```text
image: ghcr.io/sachin001234/vllm-platform:NEW_SHA
```

The update is committed to the GitOps repository.

```text
GHCR Image
    │
    ▼
GitHub Actions
    │
    ▼
vllm-gitops
    │
    ▼
Git Commit
```

---

# Cross-Repository Authentication

The CI workflow uses a GitHub Actions Secret:

```text
VLLM_GITOPS_TOKEN
```

This allows the workflow in `vllm-platform` to authenticate when updating the `vllm-gitops` repository.

The authentication flow is:

```text
vllm-platform
      │
      ▼
GitHub Actions
      │
      │ VLLM_GITOPS_TOKEN
      ▼
GitHub
      │
      ▼
vllm-gitops
```

The token is stored as a GitHub Actions Secret and is not committed to either repository.

---

# GitOps Change Detection

Once the image update is committed to `vllm-gitops`, Argo CD detects the Git change.

```text
vllm-gitops
      │
      │ New Commit
      ▼
   Argo CD
      │
      ▼
Application OutOfSync
```

Argo CD compares:

```text
Desired State in Git
        │
        ▼
Actual State in Kubernetes
```

The image difference causes the application to require synchronization.

---

# Kustomize Rendering

Argo CD processes the Kustomize configuration referenced by the Application.

For development:

```text
vllm-gitops
      │
      ▼
overlays/dev
      │
      ▼
kustomization.yaml
      │
      ▼
base/
      │
      ├── Deployment
      ├── Service
      └── Ingress
      │
      ▼
Development patches
      │
      ▼
Final Kubernetes manifests
```

Kustomize combines the base resources with the environment-specific patches to produce the desired Kubernetes configuration.

---

# Argo CD Synchronization

After detecting the Git change, Argo CD synchronizes the Kubernetes cluster.

```text
Git Change
    │
    ▼
Argo CD
    │
    ▼
Kustomize
    │
    ▼
Kubernetes Manifests
    │
    ▼
Kubernetes API
```

The Deployment is updated to reference the new container image.

```text
Old Image
    │
    ▼
Kubernetes Deployment
    │
    ▼
New Image
```

Because automated synchronization is enabled for the development Application, this process does not require a manual `kubectl apply`.

---

# Kubernetes Deployment

Kubernetes then manages the updated vLLM Deployment.

```text
Argo CD
   │
   ▼
Deployment
   │
   ▼
ReplicaSet
   │
   ▼
vLLM Pod
```

The Deployment controls the desired number of replicas.

The ReplicaSet ensures the desired number of Pods exists.

The Pod runs the vLLM container.

---

# Service and Ingress

After deployment, Kubernetes networking exposes the application through the Service and Ingress.

```text
                    Client
                      │
                      ▼
               NGINX Ingress
                      │
                      ▼
                 vLLM Service
                      │
                      ▼
                   vLLM Pod
                      │
                      ▼
                 vLLM API :8000
```

The Service provides stable internal networking.

The Ingress provides HTTP routing toward the Service.

---

# Health Checks

The vLLM Deployment uses Kubernetes health checks.

The probes use:

```text
/health
```

The flow is:

```text
Kubernetes
    │
    ├── Liveness Probe
    │
    └── Readiness Probe
             │
             ▼
        vLLM /health
```

The readiness probe determines whether the Pod should receive traffic.

The liveness probe helps Kubernetes determine whether the container needs to be restarted.

---

# Monitoring After Deployment

After the workload is deployed, monitoring observes the Kubernetes and vLLM environment.

```text
vLLM
 │
 ├── /metrics
 │
 ▼
ServiceMonitor
 │
 ▼
Prometheus
 │
 ├── Metrics
 │
 └── Alert Rules
 │
 ├───────────────┐
 ▼               ▼
Grafana      Alertmanager
                 │
                 ▼
               Slack
```

This creates an operational feedback loop after deployment.

---

# Deployment Verification

The Kubernetes deployment can be verified using:

```bash
kubectl get deployment -n vllm-dev
```

Check Pods:

```bash
kubectl get pods -n vllm-dev
```

Check the Service:

```bash
kubectl get svc -n vllm-dev
```

Check Ingress:

```bash
kubectl get ingress -n vllm-dev
```

Check the deployed image:

```bash
kubectl get deployment dev-vllm \
  -n vllm-dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Check the Argo CD Application:

```bash
kubectl get application vllm-dev -n argocd
```

---

# GitOps Deployment Status

Argo CD provides two important status concepts:

```text
Sync Status
```

and:

```text
Health Status
```

For example:

```text
Synced
```

means the Kubernetes configuration matches the desired state in Git.

The health status describes whether the actual workload is functioning correctly.

In the local environment, the vLLM Application can be:

```text
Synced
Progressing / Degraded
```

because the local Kubernetes cluster does not expose the NVIDIA GPU required by the vLLM workload.

Therefore:

```text
GitOps Synchronization
        │
        └── Working

vLLM Runtime Health
        │
        └── Limited by local GPU support
```

---

# Deployment Failure and Rollback Flow

If a bad image or configuration is committed, the GitOps system provides a controlled rollback path.

```text
Bad Commit
    │
    ▼
GitOps Repository
    │
    ▼
Argo CD
    │
    ▼
Kubernetes
    │
    ▼
Deployment Failure
    │
    ▼
git revert
    │
    ▼
Known-Good Git State
    │
    ▼
Argo CD
    │
    ▼
Kubernetes
    │
    ▼
Known-Good Deployment
```

This was tested using an intentionally invalid container image.

The resulting `ImagePullBackOff` state demonstrated the failure condition, and reverting the Git commit restored the previous image reference.

---

# Self-Healing During Deployment

Argo CD can also correct manual configuration drift.

```text
Git
 │
 └── Desired Image
          │
          ▼
      Argo CD
          │
          ▼
     Kubernetes
          │
          │ Manual Change
          ▼
       Drift
          │
          ▼
      Argo CD
          │
          ▼
   Desired State Restored
```

This provides continuous reconciliation between Git and Kubernetes.

---

# Complete Deployment Lifecycle

The entire deployment lifecycle can be summarized as:

```text
                         Developer
                             │
                             ▼
                      vllm-platform
                             │
                             ▼
                       GitHub Actions
                             │
                    ┌────────┼────────┐
                    ▼        ▼        ▼
                  Test     Build     Push
                             │
                             ▼
                            GHCR
                             │
                             ▼
                     Update GitOps
                             │
                             ▼
                       vllm-gitops
                             │
                             ▼
                          Argo CD
                             │
                    ┌────────┼────────┐
                    ▼        ▼        ▼
                  Detect   Render    Sync
                           Kustomize
                             │
                             ▼
                        Kubernetes
                             │
                    ┌────────┼─────────┐
                    ▼        ▼         ▼
                Deployment Service   Ingress
                    │
                    ▼
                  vLLM
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
      Monitoring            Health
          │                    │
          ▼                    ▼
      Prometheus          Readiness /
          │               Liveness
     ┌────┴────┐
     ▼         ▼
  Grafana   Alertmanager
                │
                ▼
              Slack
```

---

# Deployment Responsibility

The deployment process maintains a clear responsibility boundary:

```text
vllm-platform
│
├── Source / platform configuration
├── Tests
├── Docker image
└── CI/CD
        │
        ▼
      GHCR
        │
        ▼
vllm-gitops
│
├── Kubernetes desired state
├── Kustomize
├── Environments
├── Monitoring
└── Alerting
        │
        ▼
     Argo CD
        │
        ▼
   Kubernetes
        │
        ▼
      vLLM
```

This separation allows the application delivery pipeline and Kubernetes deployment configuration to evolve independently while remaining connected through the GitOps workflow.
