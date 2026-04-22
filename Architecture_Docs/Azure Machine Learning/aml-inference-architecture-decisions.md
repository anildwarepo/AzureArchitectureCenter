# Azure Machine Learning — Online & Batch Inference Reference Architecture

## Executive Summary

This document captures the architecture decisions, reasoning, and trade-offs for the Azure Machine Learning inference reference architecture. It covers multi-environment (Dev / Stage / Prod) deployment of ML models using AKS-based inference clusters with blue/green deployment, CI/CD via GitHub Actions, and private networking.

**Target audience:** Platform engineers, ML engineers, cloud architects, data scientists  
**Scope:** Online inference (AKS), batch inference (AML pipelines), CI/CD, networking, observability  
**Out of scope:** Multi-region DR, streaming inference, Foundry integration, OneLake vs ADLS Gen2

## Architecture Overview

The architecture uses three Azure subscriptions (Dev, Stage, Prod) with per-environment isolation. Each subscription contains its own AML Workspace, Container Registry, Model Registry, VNet, and AKS cluster.

**Diagrams:**
- Online inference: ![online](../../Architecture_Diagrams/Azure%20Machine%20Learning/azure-ml-managed-endpoints-multi-env.svg)
- Batch inference: ![batch](../../Architecture_Diagrams/Azure%20Machine%20Learning/azure-ml-batch-inference.svg)
- Decision points: ![decisions](../../Architecture_Diagrams/Azure%20Machine%20Learning/aks-vs-aml-native-decision-points.svg)

---

## Component Decision Records

### Inference Compute — AKS

| Attribute | Detail |
|---|---|
| **Service** | Azure Kubernetes Service (AKS) |
| **Role** | Primary inference engine for online model serving in all environments |
| **Decision** | AKS clusters in each subscription — Dev (2 inference nodes), Stage (2 nodes), Prod (8 nodes HA). Dev also has a 3-node training cluster |
| **Reasoning** | Full VM SKU flexibility (CPU + GPU, NC/ND-series), ODCR support for guaranteed GPU capacity, Kubernetes-native rolling upgrades, advanced traffic flighting via Istio/Envoy, rich observability (Prometheus/Grafana/Container Insights), team has existing K8s expertise |
| **Alternatives Considered** | AML Managed Online Endpoints — simpler but less flexible; no ODCR support, limited traffic routing, fewer GPU SKU options |
| **Trade-offs** | Higher operational complexity (cluster upgrades, node pool management, networking). Requires Kubernetes expertise. More infrastructure to manage vs fully managed endpoints |
| **Meeting Source** | Design session (Apr 13), Weekly sync (Apr 20), Work-stream sessions |

### Training Compute — AKS (Dev Only)

| Attribute | Detail |
|---|---|
| **Service** | AKS Training Cluster (3 NC/ND-series GPU nodes) |
| **Role** | Model training and experimentation — Dev environment only |
| **Decision** | Training cluster exists only in Dev. Stage and Prod are inference-only |
| **Reasoning** | Training requires GPU compute and access to raw training data. Restricting training to Dev prevents accidental data access in production, reduces cost, and simplifies Stage/Prod footprint |
| **Alternatives Considered** | AML Compute Clusters (managed) — auto-scale to zero, simpler. Decided AKS for consistency with inference and to leverage existing cluster management |
| **Trade-offs** | AKS training cluster doesn't auto-scale to zero like AML Compute. Must manually manage node pools for cost |
| **Meeting Source** | Components & networking (Mar 31), Work-stream sessions |

### Blue/Green Deployments

| Attribute | Detail |
|---|---|
| **Service** | AKS with Managed Online Endpoints (blue/green traffic split) |
| **Role** | Zero-downtime model rollouts with canary validation |
| **Decision** | Blue deployment (stable) + Green deployment (canary). Dev: 90%/10%, Stage: 100%/0%, Prod: 95%/5% |
| **Reasoning** | Enables progressive rollout — deploy new model version to green, shift small traffic percentage, validate metrics, then full cutover. Instant rollback by shifting traffic back to blue |
| **Alternatives Considered** | Kubernetes-native canary via Istio — more flexible but added complexity. AML's built-in traffic splitting is sufficient for model versioning |
| **Trade-offs** | Requires maintaining two deployments simultaneously (double resource cost during rollout). Need monitoring to detect regressions quickly |
| **Meeting Source** | Design session (Apr 13) |

### Container Registry — Per Environment

| Attribute | Detail |
|---|---|
| **Service** | Azure Container Registry (ACR) |
| **Role** | Store container images for inference serving code |
| **Decision** | One ACR per subscription, inside the AML Workspace scope |
| **Reasoning** | Data segregation — production images isolated from dev. RBAC — only CI/CD service principals can push to Stage/Prod ACR. Compliance — audit trail per environment. Prevents "works on my machine" issues with shared registries |
| **Alternatives Considered** | Shared ACR across all environments — simpler but violates data segregation requirements. Cross-subscription image pull adds networking complexity |
| **Trade-offs** | Image must be rebuilt or replicated for each environment (can't just tag and promote). Slightly higher storage cost |
| **Meeting Source** | Weekly sync (Apr 20), Work-stream sessions (registry-per-environment agreement) |

### Model Registry — Per Environment

| Attribute | Detail |
|---|---|
| **Service** | AML Model Registry |
| **Role** | Versioned catalog of trained ML models per subscription |
| **Decision** | Separate Model Registry in each environment (Dev, Stage, Prod) |
| **Reasoning** | Dev: stores freshly trained models from experiments. Stage: receives promoted models for validation. Prod: contains only approved models for serving. Enables RBAC (data scientists write to Dev, only CI/CD writes to Stage/Prod), audit trail, and blast radius isolation |
| **Alternatives Considered** | AML Registry (cross-workspace) — enables model sharing across workspaces but complicates access control and compliance tracking |
| **Trade-offs** | Model promotion requires explicit copy/register step in CI/CD. Can't browse all versions in a single view |
| **Meeting Source** | Components & networking (Mar 31), Weekly sync (Apr 20) |

### Public Ingress — Front Door + App Gateway + APIM

| Attribute | Detail |
|---|---|
| **Service** | Azure Front Door → Application Gateway (WAF) → API Management → AKS |
| **Role** | Public internet access to production inference endpoints only |
| **Decision** | Ingress chain aligned exclusively to Prod subscription. Dev and Stage have no public ingress (internal testing only) |
| **Reasoning** | Front Door: global load balancing, DDoS protection, SSL offload. App Gateway: WAF rules, SSL termination at VNet edge. APIM: rate limiting, authentication, API versioning, developer portal. Layered security model |
| **Alternatives Considered** | Direct AKS ingress controller (NGINX) — simpler but no WAF, no rate limiting. Azure Load Balancer — L4 only, no WAF |
| **Trade-offs** | Three-hop ingress adds latency (~2-5ms). Higher cost for Front Door + App GW + APIM. More complex troubleshooting |
| **Meeting Source** | Design session (Apr 13), Components & networking (Mar 31) |

### VNet — Per Environment

| Attribute | Detail |
|---|---|
| **Service** | Azure Virtual Network |
| **Role** | Network isolation for each subscription's resources |
| **Decision** | Dedicated VNet per environment. All resources (AKS, ACR, Storage, AML Workspace) connected via Private Endpoints |
| **Reasoning** | Network isolation prevents cross-environment data access. Private Endpoints ensure no public internet exposure for internal services. NSGs and network policies control east-west traffic within the VNet |
| **Alternatives Considered** | Shared VNet with subnet-per-environment — simpler but weaker isolation. Hub-spoke with peering — considered for production but adds complexity |
| **Trade-offs** | No direct connectivity between environments (by design). CI/CD runners need network access to each VNet |
| **Meeting Source** | Components & networking (Mar 31) |

### AML Workspace — Per Environment

| Attribute | Detail |
|---|---|
| **Service** | Azure Machine Learning Workspace |
| **Role** | Project-level resource management, experiment tracking, pipeline orchestration |
| **Decision** | One AML Workspace per subscription. Each workspace owns its compute, registries, and data references |
| **Reasoning** | Per-subscription model agreed in kick-off — aligns with organizational boundary (team-per-workload). Simplifies RBAC, cost tracking, and compliance |
| **Alternatives Considered** | Centralized AML Workspace shared across teams — rejected due to RBAC complexity, cost attribution difficulty, and blast radius concerns |
| **Trade-offs** | Data scientists need to context-switch between workspaces. Experiment comparison across environments requires manual effort |
| **Meeting Source** | Kick-off (Mar 5), Design session (Apr 13) |

### AML Compute Attached (AKS)

| Attribute | Detail |
|---|---|
| **Service** | AML Compute Target (attached AKS) |
| **Role** | Allows AML to schedule inference workloads on the existing AKS cluster |
| **Decision** | Present in Stage and Prod. Connects AML pipeline orchestration to AKS compute |
| **Reasoning** | Enables AML batch endpoints to run on the same AKS infrastructure used for online inference. Unifies compute management |
| **Alternatives Considered** | Separate AML Compute Clusters — simpler but adds separate infrastructure to manage |
| **Trade-offs** | AKS cluster must be sized for both online and batch workloads. Resource contention possible during batch runs |
| **Meeting Source** | Work-stream sessions |

---

## Cross-Cutting Concerns

### Networking & Security

| Decision | Detail |
|---|---|
| Private Endpoints | All resources (AKS, ACR, Storage, AML) exposed only via Private Endpoints within the VNet |
| No public endpoints | Dev and Stage have zero public ingress. Prod uses Front Door → App GW → APIM chain |
| Network policies | Kubernetes network policies restrict pod-to-pod communication |
| Managed Identities | Service-to-service auth via Managed Identities — no credentials in code |

### Observability

| Component | Tool | Purpose |
|---|---|---|
| Infrastructure metrics | Azure Monitor | Node health, CPU/GPU utilization, disk, network |
| Application performance | Application Insights | Request latency, error rates, dependency tracking |
| Log aggregation | Log Analytics | Centralized logs from AKS, AML, and infrastructure |
| Model monitoring | AML data drift | Detect training-serving skew |
| Cost tracking | Azure Cost Management | Per-environment compute spend |

### Cost Optimization

| Strategy | Detail |
|---|---|
| Spot node pools | Dev/Stage use spot instances for non-critical workloads |
| Auto-scaling | AKS cluster autoscaler scales 0→N for batch, min→max for online |
| ODCR | Prod reserves GPU capacity for SLA-bound workloads |
| Right-sizing | Node pool VM sizes matched to model requirements (CPU for lightweight, GPU for heavy) |

---

## CI/CD & Deployment Strategy

### Branching Model

| Branch | Environment | Trigger |
|---|---|---|
| `feature/*` | Dev | Data scientist pushes code |
| `integration` | Stage | PR auto-created after Dev tests pass |
| `main` | Prod | PR auto-created after Stage tests pass |

### Pipeline Flow

```
Data Scientist → push → feature/* → GH Actions Build → push to ACR → deploy to AKS (Dev)
  → automated tests pass → PR to integration → GH Actions Build → push to ACR → deploy to AKS (Stage)
    → validation pass → PR to main → GH Actions Build → push to ACR → deploy to AKS (Prod)
      → canary (5%) → gradual traffic shift → full production
```

### Deployment Details

| Step | Tool | Action |
|---|---|---|
| Build | GitHub Actions | Build container image from inference server code |
| Push | GitHub Actions → ACR | Push image to environment-specific ACR |
| Deploy | GitHub Actions → AKS | Deploy to AKS cluster, update blue/green endpoint |
| Test | Automated scripts | Unit tests (Dev), integration tests (Stage) |
| Promote | GitHub Actions | Create PR to next branch if tests pass |
| Rollback | AKS + AML | Shift traffic back to blue deployment |

---

## Data Flow

### Training Data Flow (Dev Only)

```
Training Data (Blob) → AML Training Pipeline → Training Cluster (AKS, 3 GPU nodes)
  → Trained Model → Model Registry (Dev)
Pre-trained Models (Blob) → Fine-tuning → Model Registry (Dev)
```

### Model Promotion Flow

```
Model Registry (Dev) → CI/CD promote → Model Registry (Stage) → validate → CI/CD promote → Model Registry (Prod)
```

### Inference Data Flow (All Environments)

```
Client Request → Front Door → App Gateway → APIM → AKS Inference Cluster
  → Blue/Green Endpoint → Model Inference → Response
  → Inference Data (Blob) for logging/auditing
```

---

## Gaps & Future Work

| Gap | Status | Notes |
|---|---|---|
| Multi-region production & DR | Not yet | Requires paired-region AKS with Front Door traffic manager |
| OneLake vs ADLS Gen2 | Not decided | Risk assessment needed for OneLake maturity |
| Separate networking diagram | Pending | Detailed NSG rules, subnet layout, peering topology |
| Streaming inference | Out of scope | Architecture focused on batch + online only |
| Spark batch processing | Partial | AML Compute Attached shown; dedicated Spark not depicted |
| Foundry integration roadmap | N/A | Strategic topic — depends on Microsoft roadmap |

---

## Revision History

| Date | Author | Change |
|---|---|---|
| 2026-04-22 | Architecture Team | Initial document — all decisions from 5 design meetings captured |

