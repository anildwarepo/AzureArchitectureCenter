---
name: architecture-decision-docs
description: 'Generate architecture decision documentation for Azure reference architectures. Captures component choices, reasoning, trade-offs, alternatives considered, and meeting-sourced decisions. Use when: user asks to document architecture decisions, create ADRs, write architecture rationale, explain component choices, generate architecture docs.'
---

# Architecture Decision Documentation Skill

Generate structured architecture decision documents that capture the **what**, **why**, **trade-offs**, and **alternatives** for each component in an Azure reference architecture.

## When to Use

- User asks to document architecture decisions or rationale
- User wants an Architecture Decision Record (ADR)
- User asks to explain why specific components were chosen
- User wants to create reference architecture documentation
- User asks to capture meeting decisions into docs

## Document Structure

Every architecture document MUST follow this structure:

### 1. Executive Summary
- One paragraph describing the architecture's purpose
- Target audience (platform engineers, data scientists, architects)
- Scope boundaries (what's included, what's not)

### 2. Architecture Overview
- High-level description of the system
- Reference to the SVG diagram file(s) in `./Architecture_Diagrams/{domain}/`
- Environment strategy (Dev / Stage / Prod)

### 3. Component Decision Records

For EACH major component, document:

```markdown
#### {Component Name}

| Attribute | Detail |
|---|---|
| **Service** | Azure service name |
| **Role** | What it does in this architecture |
| **Decision** | What was chosen and how it's configured |
| **Reasoning** | Why this choice was made |
| **Alternatives Considered** | What else was evaluated |
| **Trade-offs** | What you give up with this choice |
| **Meeting Source** | Which meeting/session established this decision |
```

### 4. Cross-Cutting Concerns
Document decisions that span multiple components:
- Networking & security model
- Identity & access management
- Observability strategy
- Cost optimization approach
- Compliance & governance

### 5. CI/CD & Deployment Strategy
- Branching model
- Build & deploy pipeline
- Promotion gates between environments
- Rollback strategy

### 6. Data Flow
- Training data flow (Dev only)
- Inference data flow (all environments)
- Model promotion flow (Dev → Stage → Prod)

### 7. Gaps & Future Work
- Decisions deferred or not yet made
- Known limitations
- Planned improvements

## Component Categories

When documenting an Azure ML architecture, cover these component categories:

### Compute & Inference
- Training compute (AKS, AML Compute, Spark)
- Inference compute (AKS, Managed Endpoints, Batch Endpoints)
- GPU vs CPU selection
- Node pool sizing and auto-scaling

### Networking & Ingress
- VNet design (per-environment)
- Private Endpoints
- Public ingress path (Front Door, App Gateway, APIM)
- DNS strategy

### Data & Storage
- Training data storage (Blob, ADLS Gen2)
- Inference data storage
- Model artifact storage
- Container registries (per-environment)
- Model registries (per-environment)

### CI/CD & Operations
- Source control strategy (branching model)
- Build pipeline (GitHub Actions, Azure DevOps)
- Container build & push
- Deployment automation
- Testing strategy (unit, integration, canary)

### Observability
- Monitoring (Azure Monitor)
- Application performance (App Insights)
- Log aggregation (Log Analytics)
- Data drift detection
- Cost tracking

### Security & Governance
- Identity (Managed Identities, RBAC)
- Secrets management (Key Vault)
- Network security (NSGs, Firewall)
- Compliance controls

## Decision Reasoning Patterns

Use these reasoning categories when documenting "why":

| Category | Example |
|---|---|
| **Operational maturity** | "Team has existing K8s expertise" |
| **Scale requirement** | "Need 50+ models on shared GPU infrastructure" |
| **Compliance** | "PCI-DSS requires network isolation per environment" |
| **Cost** | "Spot node pools reduce GPU costs by 60%" |
| **Simplicity** | "AML managed endpoints reduce operational burden" |
| **Flexibility** | "AKS allows custom autoscalers and traffic routing" |
| **Performance** | "ODCR guarantees GPU availability for SLA-bound workloads" |
| **Security** | "Private endpoints prevent data exfiltration" |
| **Resilience** | "Blue/green deployments enable zero-downtime rollouts" |

## Output Format

- Save architecture docs to `./Architecture_Docs/{domain}/{doc-name}.md`
- Use standard Markdown with tables
- Include relative links to SVG diagrams: `![diagram](../Architecture_Diagrams/{domain}/{file}.svg)`
- Keep each section concise — use tables over prose where possible
- Include a revision history table at the bottom

## Anti-Patterns

1. **Never document decisions without reasoning** — "We chose AKS" is incomplete. Always include WHY
2. **Never skip alternatives** — even if the choice was obvious, document what else was considered
3. **Never omit trade-offs** — every decision has downsides; document them honestly
4. **Never write without referencing source** — link decisions to specific meetings or discussions
5. **Never duplicate diagram content** — reference the SVG, don't redraw in text
