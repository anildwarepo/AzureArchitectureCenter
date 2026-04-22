# AML Reference Architecture — Requirements Backlog

> Extracted from [aml-requirements-traceability.svg](../../Architecture_Diagrams/Azure%20Machine%20Learning/aml-requirements-traceability.svg)  
> Last updated: 2026-04-22

## Summary

| Status | Count | % |
|---|---|---|
| ✅ Implemented | 19 | 83% |
| ⚠️ Partial / Not Yet / N/A | 3 | 13% |
| ❌ Not Shown | 1 | 4% |

---

## ✅ Completed Requirements

### From Meeting 1 — Design Session (Apr 13)

| ID | Requirement | Artifact | Notes |
|---|---|---|---|
| REQ-01 | Dev / Stage / Prod environment separation | `azure-ml-managed-endpoints-multi-env.svg` | 3 subscription columns with distinct colors |
| REQ-02 | AKS as inference engine (vs AML managed endpoints) | `azure-ml-managed-endpoints-multi-env.svg`, `aks-vs-aml-native-decision-points.svg` | AKS Inference Cluster in all 3 envs + decision slide |
| REQ-03 | CI/CD with GitHub Actions & git branching strategy | `azure-ml-managed-endpoints-multi-env.svg` | CI/CD row: feature → Build → ACR → AKS → integration → main |
| REQ-04 | Container registries per environment | `azure-ml-managed-endpoints-multi-env.svg` | ACR inside each subscription's workspace area |
| REQ-05 | Front Door strategy (prod-only ingress) | `azure-ml-managed-endpoints-multi-env.svg` | Public ingress aligned above Prod only, arrow pointing down |

### From Meeting 2 — Weekly Sync (Apr 20)

| ID | Requirement | Artifact | Notes |
|---|---|---|---|
| REQ-06 | Scope: online inference (no streaming) | `azure-ml-managed-endpoints-multi-env.svg` | Online endpoints shown via AKS Inference Clusters |
| REQ-07 | AKS as lead inference choice | `azure-ml-managed-endpoints-multi-env.svg` | AKS Inference Cluster primary in all envs |
| REQ-08 | Training on AML compute (Dev only) | `azure-ml-managed-endpoints-multi-env.svg` | Training Cluster (AKS) only in Dev subscription |
| REQ-09 | Registry strategy (per environment) | `azure-ml-managed-endpoints-multi-env.svg` | ACR + Model Registry inside each subscription |

### From Meeting 3 — Components & Networking (Mar 31)

| ID | Requirement | Artifact | Notes |
|---|---|---|---|
| REQ-10 | Training only in Dev | `azure-ml-managed-endpoints-multi-env.svg` | Training Cluster with 3 GPU nodes only in Dev sub |
| REQ-11 | Model promotion patterns (Dev → Stage → Prod) | `azure-ml-managed-endpoints-multi-env.svg` | CI/CD pipeline shows feature → integration → main flow |
| REQ-12 | Registry + data segregation per environment | `azure-ml-managed-endpoints-multi-env.svg` | Separate ACR, Model Registry, and Blob per subscription |
| REQ-13 | Networking & security: VNet, Private Endpoints, App GW, APIM | `azure-ml-managed-endpoints-multi-env.svg` | VNet + PE in each env; App GW → APIM → AKS in Prod |

### From Meeting 4 — Kick-off (Mar 5)

| ID | Requirement | Artifact | Notes |
|---|---|---|---|
| REQ-14 | Overall scope: AML inference reference architecture | `azure-ml-managed-endpoints-multi-env.svg` | Full multi-env AML + AKS architecture diagram |
| REQ-15 | Per-subscription deployment model (not centralized) | `azure-ml-managed-endpoints-multi-env.svg` | Each env has own subscription, workspace, ACR, VNet |

### From Meeting 5 — Work-stream (Anil, Ethan, Joakim)

| ID | Requirement | Artifact | Notes |
|---|---|---|---|
| REQ-16 | Online endpoints via AKS (primary) | `azure-ml-managed-endpoints-multi-env.svg` | AKS Inference Cluster with blue/green in all envs |
| REQ-17 | Batch inference via AML pipelines | `azure-ml-batch-inference.svg` | Separate batch inference diagram |
| REQ-18 | Public vs private inference endpoints | `azure-ml-managed-endpoints-multi-env.svg` | Public ingress via Front Door → App GW → APIM; PE in all VNets |
| REQ-19 | Registry-per-environment agreement | `azure-ml-managed-endpoints-multi-env.svg` | ACR + Model Registry in each subscription |

---

## ⚠️ Backlog — Partial / Not Yet / N/A

| ID | Requirement | Source Meeting | Priority | Status | Action Required |
|---|---|---|---|---|---|
| REQ-20 | Multi-region production & DR | Design Session (Apr 13) | High | **Not yet** | Create multi-region diagram with paired AKS clusters, Front Door traffic manager, geo-replicated ACR, and failover strategy |
| REQ-21 | Diagram structure: static / networking / process (3 views) | Weekly Sync (Apr 20) | Medium | **Partial** | Combined view done. Create separate **networking detail diagram** (NSG rules, subnet layout, peering, UDR, firewall rules) and **process flow diagram** (model lifecycle from training to retirement) |
| REQ-22 | AML + Foundry roadmap positioning | Kick-off (Mar 5) | Low | **N/A** | Strategic roadmap topic. Monitor Microsoft announcements. Document positioning once Foundry GA features are confirmed |

---

## ❌ Backlog — Not Addressed

| ID | Requirement | Source Meeting | Priority | Status | Action Required |
|---|---|---|---|---|---|
| REQ-23 | OneLake vs ADLS Gen2 risk assessment | Weekly Sync (Apr 20) | High | **Not shown** | Create comparison slide: OneLake maturity risks, ADLS Gen2 proven path, migration path if OneLake adopted later. Include data governance (Purview integration), performance benchmarks, and cost comparison |

---

## Backlog Work Items (Prioritized)

### P0 — Critical (next sprint)

| Work Item | Requirement | Deliverable | Effort |
|---|---|---|---|
| WI-01 | REQ-23: OneLake vs ADLS Gen2 | Decision slide SVG + section in architecture doc | Small |
| WI-02 | REQ-20: Multi-region DR | Multi-region architecture SVG + architecture doc section | Large |

### P1 — Important (next 2 sprints)

| Work Item | Requirement | Deliverable | Effort |
|---|---|---|---|
| WI-03 | REQ-21: Networking detail diagram | Separate SVG showing subnet layout, NSGs, UDRs, peering, firewall rules | Medium |
| WI-04 | REQ-21: Process flow diagram | Model lifecycle SVG: train → register → promote → deploy → monitor → retrain | Medium |

### P2 — Nice to have

| Work Item | Requirement | Deliverable | Effort |
|---|---|---|---|
| WI-05 | REQ-22: Foundry roadmap | Position paper / decision doc when Foundry GA features are confirmed | Small |
| WI-06 | — | Cost model spreadsheet per environment (Dev/Stage/Prod estimated monthly) | Small |
| WI-07 | — | Security & compliance checklist (PCI-DSS, SOC2 mapping to architecture controls) | Medium |

---

## Traceability Matrix

| Meeting | Date | Total Reqs | ✅ Done | ⚠️ Partial | ❌ Gap |
|---|---|---|---|---|---|
| Design Session | Apr 13 | 6 | 5 | 1 | 0 |
| Weekly Sync | Apr 20 | 6 | 4 | 1 | 1 |
| Components & Networking | Mar 31 | 4 | 4 | 0 | 0 |
| Kick-off | Mar 5 | 3 | 2 | 1 | 0 |
| Work-stream | Multiple | 5 | 5 | 0 | 0 |
| **Total** | | **24** | **20** | **3** | **1** |

