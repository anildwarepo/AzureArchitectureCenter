---
name: azure-architecture-svg
description: 'Create Azure architecture diagrams as SVG files using official Azure service icons from ./Azure_Icons/ and reference architecture patterns from ./Reference_Architecture/. Use when: user asks to create architecture diagram, draw Azure architecture, generate SVG diagram, visualize Azure solution, create reference architecture, diagram cloud infrastructure.'
---

# Azure Architecture SVG Diagram Skill

Create professional, self-contained SVG architecture diagrams using official Azure service icons and reference architecture patterns from this workspace.

## When to Use

- User asks to create, draw, or generate an Azure architecture diagram
- User asks to visualize an Azure solution or cloud infrastructure
- User references any Azure services and wants a diagram
- User mentions APIM, Fabric, OneLake, inferencing, data lake architectures

## Procedure

### Step 1 — Match Reference Architecture

Check if the user's request matches any reference file in `./Reference_Architecture/`. These contain proven Azure architecture patterns to inform your diagram.

**Available reference architectures:**

| File | Scenario | Key Components |
|---|---|---|
| `agent_architecture.svg` | AI agent system design | Agent orchestration, tools, knowledge retrieval |
| `architecture.svg` | General Azure architecture | Core Azure services and connectivity |
| `data_protection_presentation.svg` | Data protection workflows | Backup, recovery, data security |
| `data_security_jira_confluence.svg` | Data security integration | Jira, Confluence, security controls |
| `elicitation_architecture.svg` | Requirements elicitation | Elicitation workflows and data flow |
| `handoff-architecture.svg` | Agent handoff patterns | Multi-agent handoff, orchestration |
| `maf_orchestrations.svg` | Microsoft Agent Framework orchestrations | MAF workflows, agent coordination |
| `maf_superstep_execution_model.svg` | MAF superstep execution | Step-based agent execution model |
| `mcp_architecture.svg` | Model Context Protocol | MCP server/client architecture |
| `mcp_auth_tiering_diagram.svg` | MCP authentication tiers | Auth levels, security tiering |
| `slide3_streaming_architecture.svg` | Streaming data architecture | Real-time data ingestion and processing |
| `talentiq-architecture.svg` | TalentIQ platform | AI-powered talent platform architecture |

> **Note:** Reference files are SVGs that can be opened and inspected directly. Use them as layout and component guidance when the user's request matches a known pattern.

### Step 2 — Identify Azure Services and Icons

Map each component in the architecture to an official Azure service icon from `./Azure_Icons/`.

**Icon folder structure:**
```
./Azure_Icons/
  ai + machine learning/     — Azure OpenAI, Cognitive Services, Bot Services, ML
  analytics/                 — Synapse, Data Factory, Event Hubs, Databricks
  app services/              — App Services, Static Apps, App Service Plans
  compute/                   — Virtual Machines, Function Apps, AKS, Batch, VMSS
  containers/                — Kubernetes, Container Instances, Container Registries
  databases/                 — Cosmos DB, SQL Database, PostgreSQL, MySQL, Redis
  devops/                    — Azure DevOps, API Management, Application Insights
  general/                   — Resource Groups, Subscriptions, Tags, generic shapes
  identity/                  — Entra ID, Managed Identities, App Registrations
  integration/               — Logic Apps, Service Bus, Event Grid, API Management
  iot/                       — IoT Hub, IoT Edge, Digital Twins
  management + governance/   — Monitor, Advisor, Policy, Azure Arc, Automation
  networking/                — VNets, Load Balancers, Firewalls, App Gateway, Front Door
  security/                  — Key Vaults, Defender, Sentinel, Conditional Access
  storage/                   — Storage Accounts, Blob, Data Lake, NetApp Files
  web/                       — App Services, Front Door, CDN, SignalR
  other/                     — Azure VMware, Cloud Shell, miscellaneous
  new icons/                 — Recently added service icons
```

**Icon filename pattern:** `{NUMBER}-icon-service-{Service-Name}.svg`

#### Commonly Used Icons Quick Reference

| Service | Folder | Filename |
|---|---|---|
| Virtual Machine | compute | `10021-icon-service-Virtual-Machine.svg` |
| Function Apps | compute | `10029-icon-service-Function-Apps.svg` |
| App Services | app services | `10035-icon-service-App-Services.svg` |
| Kubernetes (AKS) | containers | `10023-icon-service-Kubernetes-Services.svg` |
| Container Instances | containers | `10104-icon-service-Container-Instances.svg` |
| Container Registry | containers | `10105-icon-service-Container-Registries.svg` |
| Cosmos DB | databases | `10121-icon-service-Azure-Cosmos-DB.svg` |
| SQL Database | databases | `10130-icon-service-SQL-Database.svg` |
| PostgreSQL | databases | `10131-icon-service-Azure-Database-PostgreSQL-Server.svg` |
| MySQL | databases | `10122-icon-service-Azure-Database-MySQL-Server.svg` |
| Redis Cache | databases | `10137-icon-service-Cache-Redis.svg` |
| Storage Accounts | storage | `10086-icon-service-Storage-Accounts.svg` |
| Data Lake Storage | storage | `10090-icon-service-Data-Lake-Storage-Gen1.svg` |
| Virtual Network | networking | `10061-icon-service-Virtual-Networks.svg` |
| Load Balancer | networking | `10062-icon-service-Load-Balancers.svg` |
| Application Gateway | networking | `10076-icon-service-Application-Gateways.svg` |
| Front Door / CDN | networking | `10073-icon-service-Front-Door-and-CDN-Profiles.svg` |
| Firewall | networking | `10084-icon-service-Firewalls.svg` |
| ExpressRoute | networking | `10079-icon-service-ExpressRoute-Circuits.svg` |
| VNet Gateway | networking | `10063-icon-service-Virtual-Network-Gateways.svg` |
| API Management | devops | `10042-icon-service-API-Management-Services.svg` |
| Azure DevOps | devops | `10261-icon-service-Azure-DevOps.svg` |
| Logic Apps | integration | `02631-icon-service-Logic-Apps.svg` |
| Service Bus | integration | `10836-icon-service-Azure-Service-Bus.svg` |
| Event Grid | integration | `10206-icon-service-Event-Grid-Topics.svg` |
| Event Hubs | analytics | `00039-icon-service-Event-Hubs.svg` |
| Data Factory | analytics | `10126-icon-service-Data-Factories.svg` |
| Synapse Analytics | analytics | `00606-icon-service-Azure-Synapse-Analytics.svg` |
| Databricks | analytics | `10787-icon-service-Azure-Databricks.svg` |
| Stream Analytics | analytics | `00042-icon-service-Stream-Analytics-Jobs.svg` |
| Azure OpenAI | ai + machine learning | `03438-icon-service-Azure-OpenAI.svg` |
| Cognitive Services | ai + machine learning | `10162-icon-service-Cognitive-Services.svg` |
| Cognitive Search | ai + machine learning | `10044-icon-service-Cognitive-Search.svg` |
| Bot Services | ai + machine learning | `10165-icon-service-Bot-Services.svg` |
| Machine Learning | ai + machine learning | `10166-icon-service-Machine-Learning.svg` |
| AI Studio | ai + machine learning | `03513-icon-service-AI-Studio.svg` |
| Key Vault | security | `10245-icon-service-Key-Vaults.svg` |
| Defender for Cloud | security | `10241-icon-service-Microsoft-Defender-for-Cloud.svg` |
| Sentinel | security | `10248-icon-service-Azure-Sentinel.svg` |
| Entra Domain Services | identity | `10222-icon-service-Entra-Domain-Services.svg` |
| Managed Identities | identity | `10227-icon-service-Entra-Managed-Identities.svg` |
| Entra Connect | identity | `02854-icon-service-Entra-Connect.svg` |
| App Registrations | identity | `10232-icon-service-App-Registrations.svg` |
| Monitor | management + governance | `00001-icon-service-Monitor.svg` |
| Application Insights | devops | `00012-icon-service-Application-Insights.svg` |
| Log Analytics | analytics | `00009-icon-service-Log-Analytics-Workspaces.svg` |
| Policy | management + governance | `10316-icon-service-Policy.svg` |
| Azure Arc | management + governance | `00756-icon-service-Azure-Arc.svg` |
| IoT Hub | iot | `10182-icon-service-IoT-Hub.svg` |
| Static Web Apps | app services | `01007-icon-service-Static-Apps.svg` |
| SignalR | web | `10052-icon-service-SignalR.svg` |
| Recovery Services | storage | `00017-icon-service-Recovery-Services-Vaults.svg` |
| Resource Groups | general | `10007-icon-service-Resource-Groups.svg` |
| Azure Migrate | migrate | `10281-icon-service-Azure-Migrate.svg` |
| Private Endpoints | other | `02579-icon-service-Private-Endpoints.svg` |

> If a service isn't in this table, search the `./Azure_Icons/` subfolders by service name. Icons follow the pattern `*-icon-service-{Service-Name}.svg`.

### Step 3 — Read and Embed Icon SVGs

For each Azure service in the diagram:

1. **Read the SVG file** from `./Azure_Icons/{category}/{filename}.svg`
2. **Extract inner content** — everything inside the `<svg>` tag (`<defs>`, `<path>`, `<rect>`, `<polygon>`, etc.)
3. **Prefix all IDs** — icons use GUID-based IDs for gradients/filters. Rename to avoid collisions when embedding multiple icons. Use a short prefix per icon (e.g., `vm-`, `cosmos-`, `apim-`)
4. **Update all references** — change all `url(#original-id)` and `fill="url(#original-id)"` to match the new prefixed IDs
5. **Wrap in positioned `<g>`** — use `transform="translate(X, Y) scale(S)"` to place and size the icon

#### Icon SVG Format

All Azure icons use:
- **ViewBox:** `0 0 18 18` (18×18 pixel canvas)
- **Elements:** `<path>`, `<rect>`, `<polygon>`, `<circle>` with inline gradients
- **Gradients:** `<linearGradient>` and `<radialGradient>` in `<defs>` with GUID IDs
- **Colors:** Azure blue (#0078d4), cyan (#50e6ff), with service-specific accent colors

#### Embedding Example

```xml
<!-- Virtual Machine icon at (100, 200), scaled to ~36×36px -->
<g transform="translate(100, 200) scale(2)">
  <defs>
    <linearGradient id="vm-grad1" x1="8.88" y1="12.21" x2="8.88" y2="0.21" gradientUnits="userSpaceOnUse">
      <stop offset="0" stop-color="#0078d4" />
      <stop offset="0.82" stop-color="#5ea0ef" />
    </linearGradient>
    <linearGradient id="vm-grad2" x1="8.88" y1="16.84" x2="8.88" y2="12.21" gradientUnits="userSpaceOnUse">
      <stop offset="0.15" stop-color="#ccc" />
      <stop offset="1" stop-color="#707070" />
    </linearGradient>
  </defs>
  <rect x="-0.12" y="0.21" width="18" height="12" rx="0.6" fill="url(#vm-grad1)" />
  <polygon points="11.88 4.46 11.88 7.95 8.88 9.71 8.88 6.21 11.88 4.46" fill="#50e6ff" />
  <!-- ... remaining icon elements ... -->
</g>
```

#### Scale Guidelines

| Context | Scale Factor | Rendered Size |
|---|---|---|
| Inside child boxes | `scale(1.8)` – `scale(2.2)` | ~32–40px |
| Standalone / hero | `scale(2.5)` – `scale(3)` | ~45–54px |
| Legend entries | `scale(0.9)` – `scale(1.1)` | ~16–20px |

### Step 4 — Build the SVG Diagram

Use the layout patterns and element construction templates from the [agent instructions](../../agents/azure_architecture_creator_agent_instructions.md).

**SVG skeleton:**
```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 {WIDTH} {HEIGHT}"
     font-family="Segoe UI, Arial, sans-serif">
  <defs>
    <!-- Shadow filters -->
    <filter id="shadow" x="-4%" y="-4%" width="108%" height="112%">
      <feDropShadow dx="0" dy="2" stdDeviation="3" flood-opacity="0.15"/>
    </filter>
    <filter id="shadowLg" x="-2%" y="-4%" width="104%" height="112%">
      <feDropShadow dx="0" dy="3" stdDeviation="5" flood-opacity="0.12"/>
    </filter>
    <!-- Layer gradients, arrow markers, icon gradients all go here -->
  </defs>

  <!-- Background -->
  <rect width="{WIDTH}" height="{HEIGHT}" rx="12" fill="#fafafa"/>

  <!-- Title -->
  <text x="{CENTER_X}" y="38" text-anchor="middle"
        font-size="20" font-weight="700" fill="#1e293b">{TITLE}</text>

  <!-- Architecture layers with embedded icons -->
  <!-- ... -->

  <!-- Legend -->
  <!-- ... -->
</svg>
```

**Child box with Azure icon:**
```xml
<g>
  <rect x="{X}" y="{Y}" width="{W}" height="{H}" rx="8"
        fill="{LIGHT_FILL}" stroke="{STROKE}" stroke-width="1"/>
  <!-- Azure icon centered near top of box -->
  <g transform="translate({CENTER_X - 18}, {Y + 8}) scale(2)">
    <!-- embedded icon SVG content with prefixed IDs -->
  </g>
  <!-- Service name below icon -->
  <text x="{CENTER_X}" y="{Y + 52}" text-anchor="middle"
        fill="{DARK_TEXT}" font-size="12" font-weight="600">{SERVICE_NAME}</text>
  <text x="{CENTER_X}" y="{Y + 66}" text-anchor="middle"
        fill="{MED_TEXT}" font-size="10">{SUBTITLE}</text>
</g>
```

**Numbered step annotations (from reference architectures):**
```xml
<!-- Step number circle -->
<circle cx="{X}" cy="{Y}" r="12" fill="#0078d4"/>
<text x="{X}" y="{Y + 4}" text-anchor="middle"
      font-size="11" font-weight="bold" fill="#fff">{N}</text>
<!-- Step description nearby -->
<text x="{X + 20}" y="{Y + 4}" font-size="10" fill="#334155">{DESCRIPTION}</text>
```

**Zone containers (On-Prem / Cloud boundary):**
```xml
<g>
  <rect x="{X}" y="{Y}" width="{W}" height="{H}" rx="10"
        fill="#f8fafc" stroke="#94a3b8" stroke-width="1.5" stroke-dasharray="8,4"/>
  <text x="{X + 10}" y="{Y + 18}" font-size="12" font-weight="700"
        fill="#475569">{ZONE_NAME}</text>
</g>
```

### Step 5 — Color System

| Semantic Role | Header Gradient | Fill | Stroke | Text |
|---|---|---|---|---|
| Frontend / UI | `#059669` | `#ecfdf5` | `#10b981` | `#065f46` |
| Agents / AI | `#7c3aed` | `#f5f3ff` | `#8b5cf6` | `#5b21b6` |
| API Gateway | `#0ea5e9` | `#f0f9ff` | `#0284c7` | `#0369a1` |
| Backend / Data | `#d97706` | `#fffbeb` | `#fbbf24` | `#78350f` |
| Infrastructure | `#2563eb` | `#dbeafe` | `#60a5fa` | `#1e3a8a` |
| On-Premises | `#64748b` | `#f1f5f9` | `#94a3b8` | `#334155` |
| Security / Auth | `#4f46e5` | `#eef2ff` | `#818cf8` | `#3730a3` |
| Monitoring | `#e74c9e` | `#fdf2f8` | `#ec4899` | `#9d174d` |
| Messaging | `#0d9488` | `#f0fdfa` | `#5eead4` | `#134e4a` |

Use max 6–7 colors per diagram.

### Step 6 — Validate and Save

**Quality checklist:**
- [ ] Every Azure service uses its official icon from `./Azure_Icons/` (no emoji for Azure services)
- [ ] All embedded icon gradient IDs are uniquely prefixed (no collisions)
- [ ] Icons are properly scaled and centered within boxes
- [ ] Font sizes: title=20, headers=14, box titles=12, subtexts=10 (never below 10)
- [ ] No text or labels overlap other text, icons, or box boundaries
- [ ] Nested containers (VNet inside subscription) are sized bottom-up to fit all children with 15px padding
- [ ] At least 15–20px vertical gap between sibling elements at the same nesting level
- [ ] Icon + label pairs budget 40–50px vertical space below the preceding element
- [ ] All `<defs>` references (`url(#...)`, `marker-end`) resolve correctly
- [ ] No text overlap; arrows don't cross text
- [ ] Legend covers all visual categories
- [ ] `viewBox` accommodates all content with 40–60px margins
- [ ] SVG is valid XML — all tags closed, attributes quoted
- [ ] No `<style>`, `<script>`, or `<image href>` — fully self-contained

**Save to:** `./Architecture_Diagrams/{descriptive-name}.svg`

## Anti-Patterns

1. **Never use emoji for Azure services** — always embed the official SVG icon
2. **Never use `<image href="...">`** — embed icon SVG content inline
3. **Never reuse gradient IDs across icons** — prefix each icon's IDs uniquely
4. **Never use font-size below 10** — unreadable when exported
5. **Never hardcode `width`/`height` on `<svg>`** — use `viewBox` only
6. **Never skip the legend** — every diagram needs one
7. **Never overlap boxes or text** — maintain 20–30px spacing minimum
8. **Never leave orphan arrows** — every arrow connects two visible elements

## Non-Azure Components

For components that are NOT Azure services (end users, on-prem servers, third-party tools), use Unicode emoji:
- 🏢 On-premises server, 🌐 Web browser, 📱 Mobile app, 💬 Chat client, 👤 End user
- 🔄 External workflow, 📋 Document, ⚡ External trigger
