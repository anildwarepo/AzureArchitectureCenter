# Azure Machine Learning — Online & Batch Inference Reference Architecture

## Executive Summary

This document captures the architecture decisions, reasoning, and trade-offs for the Azure Machine Learning inference reference architecture. It covers multi-environment (Dev / Stage / Prod) deployment of ML models using AKS-based inference clusters with blue/green deployment, CI/CD via GitHub Actions, and private networking.

**Target audience:** Platform engineers, ML engineers, cloud architects, data scientists  
**Scope:** Online inference (AKS), batch inference (AML pipelines), CI/CD, networking, observability, Fabric/OneLake data sources, Feature Store (online + offline)  
**Out of scope:** Multi-region DR, streaming inference, Foundry integration

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

### Training Compute — AML Training Compute Cluster (Dev Only)

| Attribute | Detail |
|---|---|
| **Service** | AML Training Compute Cluster (managed Azure ML compute, NC/ND-series GPU) |
| **Role** | Model training and experimentation — Dev environment only |
| **Decision** | Training runs on a managed **AML Compute Cluster** (3 GPU nodes, auto-scale-to-zero) inside the Dev workspace. Training does **not** run on AKS. AKS is reserved for online inference. Training cluster exists only in Dev; Stage and Prod are inference-only |
| **Reasoning** | AML Compute Cluster is purpose-built for ML training: native integration with AML Datasets / Datastores / Jobs, auto-scale-to-zero (so the cluster costs nothing when idle), low-priority VM support, first-class MLflow + lineage tracking, and no Kubernetes operational overhead. Splitting training off AKS keeps the inference cluster predictable for SLOs and avoids resource contention between long GPU training jobs and latency-sensitive scoring |
| **Alternatives Considered** | AKS training cluster (previous design) — rejected: doesn't auto-scale to zero, requires K8s plumbing for every job, mixes training noise with inference SLOs, and duplicates the scheduling capability AML already provides. AML Serverless Compute — viable for ad-hoc but lacks the dedicated quota / ODCR story for sustained training campaigns |
| **Trade-offs** | Two compute paradigms in the platform (AML-managed for training, AKS for inference). Training cluster image management is AML-curated rather than fully custom. Spot/low-priority on AML Compute means jobs may be preempted — fine for training, must be checkpointed |
| **Meeting Source** | ML platform working session (May), Components & networking (Mar 31) |

### Why AML for Training (not Spark in Fabric)

| Attribute | Detail |
|---|---|
| **Service** | Azure Machine Learning (Training Compute Cluster + AML SDK v2) |
| **Role** | Execution surface for model training jobs orchestrated by Fabric |
| **Decision** | Training jobs run on **AML Training Compute Cluster** invoked via the **AML SDK v2 / CLI v2**. Fabric Spark notebooks are **not** used for model training, only for upstream data preparation in OneLake |
| **Reasoning** | Our data scientists are not Spark-heavy. They write standard Python (PyTorch / scikit-learn / XGBoost / Hugging Face) and need a lightweight, opinionated SDK rather than a distributed compute paradigm. AML SDK v2 gives them, out of the box: **Dataset versioning** (immutable training inputs), **Model versioning** in the AML Model Registry, **Feature Store** integration, **parallel jobs** (sweep, hyperparameter tuning, distributed training across nodes), **AutoML**, **MLflow tracking** and **end-to-end lineage** from dataset version → run → model → endpoint. Fabric handles orchestration upstream; AML owns the training run itself |
| **Alternatives Considered** | Fabric Spark notebooks for training — rejected: forces every data scientist to learn Spark for jobs that fit comfortably on a single GPU node, no native model-versioning / Feature Store / AutoML story, and lineage tracking is weaker for ML artifacts. Databricks ML — viable but adds a third platform alongside Fabric and AML |
| **Trade-offs** | Two SDKs in the platform: Fabric for data engineers, AML SDK v2 for ML engineers. Truly Spark-scale training (hundreds of nodes, terabyte-scale shuffles) would require a different runtime — not currently a workload we have |
| **Meeting Source** | ML platform working session (May) |

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

### Microsoft Fabric — External Data & Orchestration Environment

| Attribute | Detail |
|---|---|
| **Service** | Microsoft Fabric (Fabric Pipelines / Data Factory + scheduler) |
| **Role** | External data + orchestration plane that lives **outside** the AML subscriptions. Owns ingest, curation, feature build, and the orchestration of every AML training job (initial and continuous) |
| **Decision** | Fabric is treated as a separate environment / component (not co-located with any AML workspace). It sits to the **left** of the Dev subscription in the architecture and integrates with AML over REST + Managed Identity. Fabric Pipelines kick off AML training jobs; AML never schedules itself |
| **Reasoning** | Decouples the data + orchestration plane from the ML compute plane: Fabric is owned by Data Engineering and serves multiple downstream consumers (Power BI, AML, ad-hoc analytics), while AML workspaces are owned per-ML-workload team. Putting Fabric outside the AML subscription boundary makes ownership, billing, and RBAC cleanly separable, and reflects how the org actually operates |
| **Alternatives Considered** | AML Pipelines as the only orchestrator — forces ML teams to also own data-engineering plumbing. Co-locate Fabric inside each AML subscription — multiplies cost and breaks the single-source-of-truth model |
| **Trade-offs** | Adds a cross-tenant integration (Fabric → AML) that must be authenticated with Managed Identity + RBAC. Fabric capacity SKU is a separate cost line. Failures span two control planes, requiring federated alerting |
| **Meeting Source** | Data & orchestration working session, Weekly sync (May) |

### OneLake — Data Source Referenced by AML Datasets

| Attribute | Detail |
|---|---|
| **Service** | OneLake (Fabric lakehouse, Delta tables) consumed via **AML Datastore + AML Datasets** |
| **Role** | Authoritative lakehouse for raw + curated training data. AML never copies the data; it references it through an AML Datastore that points at a OneLake shortcut, and exposes versioned snapshots as AML Datasets |
| **Decision** | OneLake is the primary training data source. Inside the AML workspace, an **AML Datastore** is registered against the OneLake URI (or via a OneLake shortcut into ADLS), and **AML Datasets** are versioned references used by training jobs. No bulk copy into the AML workspace storage |
| **Reasoning** | Single copy of the data — Fabric, Power BI, and AML all read the same Delta files. AML Datasets give versioning + lineage so every training run is reproducible against an exact data snapshot. Avoids the duplicate-data tax and the drift risk of dual-write |
| **Alternatives Considered** | Bulk copy OneLake → workspace Blob — simple but doubles storage cost and creates a stale-data risk. Direct OneLake access from training code (no AML Dataset wrapper) — loses versioning and lineage |
| **Trade-offs** | OneLake connectors in AML are still maturing; some scenarios use a OneLake shortcut into ADLS Gen2 as a bridge. Cross-region access requires explicit design |
| **Meeting Source** | Data & orchestration working session |

### Continuous Training (MLOps Loop)

| Attribute | Detail |
|---|---|
| **Service** | Fabric Pipelines + AML Jobs + AML Model Registry + AML monitoring |
| **Role** | Closed-loop retraining triggered by data drift, feature freshness, or scheduled cadence |
| **Decision** | Fabric Pipelines own the orchestration of both **initial training** and the **continuous training loop**. Loop steps: (1) AML monitoring / drift signals or scheduled trigger fire → (2) Fabric pipeline runs → (3) refresh OneLake feature tables → (4) submit AML training job on the AML Training Compute Cluster → (5) register the new model version in AML Model Registry (Dev) → (6) CI/CD picks it up and promotes through Stage → Prod |
| **Reasoning** | Putting orchestration in Fabric means the same engine runs initial training and re-training — no second orchestrator to maintain. Centralising the loop in Fabric also lets data engineers own data freshness end-to-end without bouncing into AML |
| **Alternatives Considered** | AML Pipelines as the loop owner — fragments orchestration across two systems. Event Grid → Functions → AML — more moving parts than Fabric provides natively |
| **Trade-offs** | Tight coupling between Fabric availability and retraining cadence — mitigated by manual fallback (AML CLI). Drift signals must be exported from AML monitoring into Fabric (Event Grid or polled) |
| **Meeting Source** | Data & orchestration working session (May) |

### Feature Store (Online + Offline) — Across Training & Inference

| Attribute | Detail |
|---|---|
| **Service** | Feature Store — Offline store on OneLake/ADLS (Delta), Online store on low-latency KV (Azure Cache for Redis Enterprise or Cosmos DB) |
| **Role** | Single, governed source of features used both at training time (offline) and at inference time (online), guaranteeing train/serve parity |
| **Decision** | Stand up a managed Feature Store with two tiers: **Offline** store backs Training Cluster jobs in Dev (and validation in Stage); **Online** store is read by the Inference Cluster in every environment for sub-10ms feature lookups during scoring |
| **Reasoning** | Eliminates training-serving skew (the #1 source of silent model regressions), enables feature reuse across models, and centralises feature lineage + freshness SLAs. Online/Offline split lets each tier be sized for its access pattern (throughput vs. latency) |
| **Alternatives Considered** | Per-model ad-hoc feature engineering inside the inference container — fast to ship, but every model re-implements the same features and skew is impossible to detect. AML Managed Feature Store — viable but currently lacks some required online-store SKUs and per-feature RBAC |
| **Trade-offs** | Two stores to operate (offline + online) and a sync pipeline between them. Adds latency-sensitive infrastructure (Redis/Cosmos) to the inference critical path. Schema-evolution discipline becomes mandatory |
| **Meeting Source** | ML platform working session, Weekly sync |

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
| Right-sizing | Node pool VM sizes matched to model requirements (CPU for lightweight, GPU for heavy) |
| Auto-pause Fabric capacity | Pause Fabric capacity outside business hours in Dev to cap orchestration cost |
| Feature reuse | Feature Store amortises feature-engineering cost across multiple models |

### Reliability

| Strategy | Detail |
|---|---|
| Highly available AKS | Prod inference cluster runs 8 nodes across availability zones with surge upgrades |
| Blue/Green endpoints | Instant rollback by shifting 100% traffic back to the stable (blue) deployment |
| **ODCR (On-Demand Capacity Reservation)** | Prod reserves GPU capacity (NC/ND-series) so production inference is never starved during regional capacity crunches — *moved here from Cost Optimization because the primary driver is SLA protection, not cost* |
| Online Feature Store HA | Redis Enterprise / Cosmos DB with zone redundancy so feature lookups never become the single point of failure on the inference path |
| Health probes & PDBs | Kubernetes liveness/readiness probes plus PodDisruptionBudgets keep min replicas during node upgrades |
| Backpressure & circuit breakers | APIM rate limits + AKS HPA absorb traffic spikes; circuit breakers fail fast on dependency loss |

### Performance Efficiency

| Strategy | Detail |
|---|---|
| GPU SKU selection | NC/ND-series matched per-model (training vs. inference profile); right-size to avoid GPU over-provisioning |
| Online Feature Store | Sub-10ms feature lookup keeps p99 inference latency inside SLO without per-request feature recomputation |
| Horizontal pod autoscaler | AKS HPA scales inference pods on QPS + GPU utilisation, not just CPU |
| Caching | APIM response cache for idempotent scoring requests; KV cache for repeat embeddings |
| Co-located data | OneLake + Training Cluster + Feature Store offline tier in the same region to eliminate cross-region read latency |
| Async batch lane | Long-running batch inference uses AML batch endpoints on attached AKS — keeps the synchronous online endpoint hot for low-latency traffic |

---

## CI/CD & Deployment Strategy

### Branching Model

| Branch | Environment | Trigger |
|---|---|---|
| `feature/*` | Dev | Data scientist pushes code |
| `integration` | Stage | PR auto-created after Dev tests pass |
| `main` | Prod | PR auto-created after Stage tests pass; **PR is merged into `main` only after the Prod deploy succeeds** |

### Pipeline Flow

```
Data Scientist → push → feature/* → GH Actions Build → push to ACR → deploy to AKS (Dev)
  → automated tests pass → PR to integration → GH Actions Build → push to ACR → deploy to AKS (Stage)
    → validation pass → PR to main → GH Actions Build → push to ACR → deploy to AKS (Prod)
      → canary (5%) → gradual traffic shift → full production
        → on success: merge PR into main
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

![Training Data Flow](../../Architecture_Diagrams/Azure%20Machine%20Learning/aml-training-data-flow.svg)

### Continuous Training Loop (Closed-Loop MLOps)

![Continuous Training Loop](../../Architecture_Diagrams/Azure%20Machine%20Learning/aml-continuous-training-loop.svg)

### Model Promotion Flow

```
Model Registry (Dev) → CI/CD promote → Model Registry (Stage) → validate → CI/CD promote → Model Registry (Prod)
```

### Inference Data Flow (All Environments)

```
Client Request → Front Door → App Gateway → APIM → AKS Inference Cluster
  → Blue/Green Endpoint
        ├── Feature Store (Online)  ── lookup features ──┐
        └── Model Inference  ◄──────────────────────────┘
  → Response
  → Inference Data (Blob) for logging/auditing
  → Feature Store (Offline)  ← async write-back of materialised features for replay
```

---

## Gaps & Future Work

| Gap | Status | Notes |
|---|---|---|
| Multi-region production & DR | Not yet | Requires paired-region AKS with Front Door traffic manager |
| OneLake vs ADLS Gen2 | **Decided** | OneLake is primary; Blob retained only for legacy raw + pre-trained model artifacts |
| Separate networking diagram | Pending | Detailed NSG rules, subnet layout, peering topology |
| Streaming inference | Out of scope | Architecture focused on batch + online only |
| Spark batch processing | Partial | Fabric notebooks + AML Compute Attached cover most cases; dedicated Spark cluster not depicted |
| Foundry integration roadmap | N/A | Strategic topic — depends on Microsoft roadmap |
| Feature Store online tier SKU | Pending | Redis Enterprise vs. Cosmos DB final selection (latency vs. multi-region failover) |
| Fabric ↔ AML auth pattern | Pending | Standardise on Managed Identity + RBAC; document Service Principal fallback for break-glass |

---

## Revision History

| Date | Author | Change |
|---|---|---|
| 2026-04-22 | Architecture Team | Initial document — all decisions from 5 design meetings captured |
| 2026-05-04 | Architecture Team | Added Microsoft Fabric (orchestration) and OneLake (data source) on the training plane; introduced Feature Store (Online + Offline) across training and inference; added **Reliability** and **Performance Efficiency** pillars; moved **ODCR** from Cost Optimization to Reliability (driver is SLA protection, not cost) |
| 2026-05-04 | Architecture Team | Training compute switched from AKS to **AML Training Compute Cluster** (managed, auto-scale-to-zero); **Microsoft Fabric repositioned as an external environment** outside the AML subscriptions (left of Dev); **OneLake referenced via AML Datastore + AML Datasets** (no copy); documented the **Continuous Training (MLOps) loop** owned by Fabric Pipelines |
| 2026-05-04 | Architecture Team | Added **AML Training vs. Spark in Fabric** decision record (lightweight AML SDK v2 chosen for non-Spark-heavy data scientists; covers dataset versioning, model versioning, Feature Store, parallel jobs, AutoML, full lineage); documented the explicit **merge PR → main** step that closes the CI/CD pipeline only after a successful Prod deploy |

