# Workstream C: Data Architecture and Pipeline Enablement — Detailed Methodology

## 1.0 Approach Overview

This document describes the detailed methodology KPMG will employ to design and implement scalable Bronze-to-Silver-to-Gold pipelines, establish the enterprise canonical data model, and enable governed self-service analytics within the Databricks Lakehouse. It supplements SOW Section 2.5 (Workstream C) by providing the specific canonical model design framework, pipeline template engineering approach, data product promotion criteria, governance implementation patterns, and Gold workspace enablement strategy.

**The Problem:**
CVS has Bronze data landing in Databricks (via WS-B2) but no governed transformation layer. Without Silver normalization, every consumer writes ad-hoc parsing logic — duplicating effort across teams, creating inconsistent reporting, and blocking AI readiness. Without a canonical data model, security and observability analytics remain siloed. Without governed Gold workspaces, 10,000 users have no path from Splunk to Databricks.

**The Objective:**
Build the transformation and analytics infrastructure that turns raw Bronze telemetry into trusted, reusable enterprise data products. Specifically: propose and establish a canonical data model with source→model→ontology mapping; construct 18–22 reusable DLT pipeline templates that instantiate into 240–320 Silver tables across 9 business domains; implement governance (PHI/PII masking, row-level security, HIPAA audit trails) via Unity Catalog; define 15 conformed dimensions for cross-domain analytics; and enable team-owned Gold workspaces with compute governance, Genie configuration, and self-service starter content.

**Delivery Model:**
WS-C operates with 4 joint KPMG/SciCom engineering pods, each responsible for ~2 Silver business domains. The Build phase establishes templates, governance patterns, and the first domain implementations. The Operate phase scales across remaining domains using the factory model.

**Relationship to Other Workstreams:**
- Workstream B1 provides data contracts (schema, ownership, classification, SLAs) that C consumes as pipeline input specifications
- Workstream B2 provides populated Bronze tables that C transforms into Silver
- Workstream D consumes Silver/Gold tables as migration targets for SPL-converted queries and dashboards
- Workstream E coordinates user access provisioning and training for Gold workspaces

**Architecture Baseline:**
This methodology references the Silver/Gold architecture defined in the Technical Implementation Approach (Scicom, v1.0, Jul 26, 2026 — Sections 8–9) and KPMG's Databricks design recommendations as the design baseline. Specifically:
- SciCom's Bronze schema standards (Section 7.3) and canonical metadata model define the physical ingestion contract
- SciCom's Bronze-to-Silver transformation contracts (Section 7.10) define the interface between ingestion and pipeline engineering
- SciCom's 9 Silver business domains (Section 8.4) and ~68–150 Silver tables establish the target domain architecture
- SciCom's 12 Gold Business Products (Section 9.3) define the consumption layer

**Table Count Reconciliation — KPMG Templates vs. SciCom Domain Tables:**
KPMG's pipeline methodology produces 18–22 DLT templates × 12–15 instantiations = 240–320 Silver table instantiations. SciCom's architecture specifies 9 Silver domains with 68–150 physical Silver tables. These figures represent different abstraction levels:
- **KPMG's 240–320:** Total pipeline instantiations — one per source family passing through a template. Each instantiation produces a Silver output, but multiple instantiations may feed the same logical Silver domain table (e.g., Cisco, PAN, F5 each instantiate from the "Security — OCSF Canonical" template, all feeding `silver.security_network_sessions`).
- **SciCom's 68–150:** Logical domain tables — the actual queryable Silver tables that consumers (WS-D, Gold products, BI tools) reference. Multiple pipeline instantiations converge into each domain table.

The relationship is: **Many pipeline instantiations (240–320) → fewer physical Silver tables (68–150) → 9 Silver business domains → 12 Gold products**. This is by design — the template model scales independently of the domain table count.

**Engagement Scope Boundary:**
The full Silver/Gold pipeline infrastructure requires approximately 1,000 Bronze-to-Silver pipelines across all source families. The current engagement delivers the template engineering framework (18–22 DLT templates) and instantiates priority pipelines aligned to available capacity and the highest-value source families identified by WS-B1. First-principles effort analysis indicates current engagement capacity supports approximately 350–400 priority pipelines within the contracted timeline and budget. The remaining pipeline volume is addressed through the factory model's scalability — additional instantiations use the same proven templates and are handled via subsequent phases or Change Order (SOW Section 9). This phased approach ensures CVS receives a production-ready, scalable pipeline infrastructure that can be extended beyond the initial engagement without rework.

---

## 2.0 Canonical Data Model and Ontology Strategy

### 2.1 Why a Canonical Model Is Required

Chris Robinson's feedback (Jun 25) identified the absence of a canonical data model as a critical gap: "wants unified schema across security + observability. OCSF is security-only. Need broader strategy." Josh Sparaga's email (Jul 28) specifically requested a "source→model→ontology mapping strategy."

The challenge: security telemetry benefits from full canonical normalization (OCSF/ECS) for cross-domain detection, while observability telemetry is often consumed in its native schema by domain-specific teams. A single normalization approach does not fit both.

### 2.2 Tiered Canonical Approach

The canonical data model uses a three-tier strategy that balances normalization depth against consumer needs:

```
Tier 1 — Full Canonical (OCSF/ECS)
    Security sources requiring cross-domain detection, correlation, and compliance
    Examples: firewall, endpoint, identity, cloud security, threat intel

Tier 2 — Governed Native Schema
    Observability sources consumed by domain-specific teams without cross-domain joins
    Examples: VMware performance, Pharmacy transactions, K8s application logs

Tier 3 — Common Metadata Envelope
    ALL sources — regardless of Tier 1 or 2 — share a common metadata envelope
    for enterprise-wide discoverability, lineage, and governance
```

| Tier | Normalization Depth | Schema Approach | Use Case | Example Sources |
|---|---|---|---|---|
| **1 — Full Canonical** | Deep | OCSF/ECS normalized fields; vendor-neutral; common event taxonomy | Cross-domain security detection, compliance, SOC dashboards, threat hunting | Cisco, PAN, F5, CrowdStrike, Windows Security, Azure AD, Cloud audit |
| **2 — Governed Native** | Medium | Domain-specific schema; governed field names and types; no forced canonical mapping | Domain-specific operations, performance monitoring, application analytics | VMware, K8s app logs, Pharmacy IVR, RxConnect, DataPower, databases |
| **3 — Metadata Envelope** | Light | Common metadata fields on ALL tables (both Tier 1 and 2) | Enterprise discoverability, governance, lineage, AI semantic search | All sources — envelope wraps both Tier 1 and Tier 2 |

### 2.3 Source → Model → Ontology Mapping

The mapping flows through three stages, each adding business meaning:

```
SOURCE (B1 Data Contracts)
    Source families, raw schemas, ownership, classification
    ↓
MODEL (Silver Business Domain Tables)
    Normalized schemas (OCSF for Tier 1; governed native for Tier 2)
    Common metadata envelope on all tables
    Conformed dimensions for cross-domain joins
    ↓
ONTOLOGY (Gold Semantic Products + Genie Layer)
    Business KPIs, executive metrics, AI feature sets
    Genie semantic models enabling natural language queries
    Databricks AI/ML feature store
```

**Stage 1 — Source → Model (Bronze → Silver):**

| Input | Transformation | Output |
|---|---|---|
| B1 data contract (schema, classification, SLA) | DLT pipeline applies parsing, normalization, enrichment, quality rules | Silver domain table (Tier 1 canonical or Tier 2 governed native) |
| B1 security classification (PHI/PII/PCI) | Masking pipeline strips or tokenizes regulated fields | Silver table with classification: INTERNAL (down from RESTRICTED at Bronze) |
| B1 source family metadata | Promoted to Silver columns; joined with conformed dimensions | Enriched Silver record with application, business unit, environment context |

**Stage 2 — Model → Ontology (Silver → Gold):**

| Input | Transformation | Output |
|---|---|---|
| Silver domain tables (normalized) | Aggregation, KPI computation, cross-domain joins | Gold analytical product (e.g., gold.executive_enterprise_health) |
| Silver + conformed dimensions | Semantic modeling for business consumption | Genie semantic model (e.g., genie.pharmacy, genie.security_operations) |
| Silver + historical data | Feature engineering, time-series aggregation | AI feature store (e.g., gold.ai_anomaly_detection, gold.ai_capacity_prediction) |

### 2.4 Conformed Dimensions

15 shared enterprise dimensions provide consistent joins across all Silver domains and Gold products:

| Dimension | Purpose | Source |
|---|---|---|
| `dim_application` | Enterprise application catalog | CMDB / ServiceNow |
| `dim_business_unit` | Organizational hierarchy | HR / Org directory |
| `dim_environment` | DEV / QA / UAT / PROD | Deployment metadata |
| `dim_cloud_provider` | Azure / AWS / GCP | Cloud APIs |
| `dim_region` | Geographic region or data center | Network topology |
| `dim_datacenter` | On-premise facility identifier | Facility registry |
| `dim_asset` | Enterprise infrastructure asset inventory | CMDB |
| `dim_hostname` | Host inventory (servers, VMs, containers) | Discovery tools |
| `dim_cluster` | Kubernetes and VMware cluster registry | Platform metadata |
| `dim_user` | Enterprise identity (employees, service accounts) | Entra ID / IAM |
| `dim_vendor` | Technology vendor catalog | Procurement |
| `dim_service` | Business service hierarchy | ServiceNow CMDB |
| `dim_severity` | Standardized severity model (0–5) | Internal standard |
| `dim_time` | Enterprise calendar (business days, holidays, quarters) | Static reference |
| `dim_configuration_item` | CMDB configuration item reference | ServiceNow |

**Slowly Changing Dimension Strategy:**

| Dimension | SCD Type | Rationale |
|---|---|---|
| Applications, Business Services, Assets, Configuration Items, Business Units, Cloud Resources | Type 2 | Preserve historical context for reporting and AI models |
| Severity Reference, Time | Type 1 / Static | Reference data that does not require history |

---

## 3.0 Pipeline Template Engineering

### 3.1 Template-Driven Scale

WS-C does not build a unique pipeline for every source. Instead, it constructs 18–22 parameterized DLT pipeline templates that are instantiated per source family using metadata from B1 data contracts.

```
18–22 DLT Templates × ~12–15 instantiations each = ~240–320 Silver Tables
```

Each template encodes a common transformation pattern. Source-specific behavior is driven by configuration parameters (schema mapping, quality thresholds, masking rules) rather than custom code.

### 3.2 Template Categories

| Template Category | Est. Templates | Covers | Key Transformation |
|---|---|---|---|
| **Security — OCSF Canonical** | 3–4 | Network, endpoint, identity, cloud security | Full OCSF normalization; vendor-neutral fields; MITRE ATT&CK mapping |
| **Infrastructure — Performance** | 2–3 | VMware, Windows, Linux, hardware | Metric normalization; resource utilization; capacity aggregation |
| **Cloud — Multi-Provider** | 2–3 | Azure, AWS, GCP activity/audit/resource | Provider-neutral cloud operations model; FinOps metrics |
| **Application — K8s** | 2–3 | K8s application logs, platform events | Container/pod/deployment metadata promotion; log parsing |
| **Application — Enterprise** | 2–3 | Pharmacy, Clinical, DataPower, Web/Middleware | Domain-specific parsing; business transaction extraction |
| **Messaging — Event Streaming** | 1–2 | Kafka, MQ, Event Hub, Pub/Sub | Message flow analytics; queue metrics |
| **Audit — Compliance** | 1–2 | Linux audit, cloud audit, access logs | Compliance evidence formatting; retention enforcement |
| **Reference — Dimension** | 1–2 | CMDB, ServiceNow, identity, asset registry | SCD Type 2 merge; deduplication; cross-reference validation |

### 3.3 Template Architecture

Each DLT pipeline template follows a consistent architecture:

```python
# Parameterized DLT Pipeline Template (pseudocode)

@dlt.table(name="{silver_table_name}")
@dlt.expect_all_or_drop(quality_rules_from_contract)
def silver_transform():
    bronze = dlt.read_stream("{bronze_table}")

    # 1. Schema Validation — reject malformed events
    validated = validate_schema(bronze, contract.expected_schema)

    # 2. Parsing — extract structured fields from raw_payload
    parsed = parse_payload(validated, contract.parse_rules)

    # 3. Normalization — apply canonical mapping (OCSF for Tier 1, native for Tier 2)
    normalized = apply_normalization(parsed, contract.normalization_tier)

    # 4. Enrichment — join with conformed dimensions
    enriched = enrich(normalized, dimensions=[dim_application, dim_asset, dim_hostname])

    # 5. Masking — apply PHI/PII masking rules from contract
    masked = apply_masking(enriched, contract.masking_rules)

    # 6. Deduplication — remove duplicate events
    deduped = deduplicate(masked, keys=contract.dedup_keys, window=contract.dedup_window)

    # 7. Quality Expectations — validate completeness, freshness, null thresholds
    return deduped
```

### 3.4 DLT Tiering Model

Not all pipelines need to run continuously. Tiering by latency requirement reduces compute cost by 60–70%:

| Tier | DLT Mode | Trigger | Latency | Use Case | Est. % of Pipelines |
|---|---|---|---|---|---|
| **Hot** | Continuous streaming | Always-on | Seconds–minutes | SIEM-feeding, real-time detection, SOC dashboards | ~15% |
| **Warm** | Triggered micro-batch | Every 15 min | 15–30 min | Compliance, audit, operational reporting | ~35% |
| **Cold** | Scheduled batch | Daily | Hours | Historical analysis, capacity planning, long-tail sources | ~50% |

Tier assignment is driven by the data contract SLA (from B1) and downstream consumer requirements (from D).

### 3.5 AI-Assisted Pipeline Instantiation

For each new source family with an approved B1 data contract:

1. **Schema inference:** AI agent reads sample Bronze data and proposes Silver schema mapping
2. **Template matching:** AI selects the appropriate DLT template category based on source characteristics
3. **Configuration generation:** AI generates template parameters (parse rules, normalization mapping, quality thresholds, masking rules) from the B1 contract
4. **Human review:** KPMG engineer validates generated configuration before deployment
5. **Test execution:** Pipeline runs against sample data; output compared to expected schema
6. **Production deployment:** Pipeline promoted to production via CI/CD (Git-based Databricks Workflows)

This approach enables onboarding velocity beyond linear headcount: the AI handles the repetitive parameterization while engineers focus on quality assurance and edge cases.

---

## 4.0 Silver Domain Implementation

### 4.1 Nine Enterprise Silver Domains

Silver tables are organized into 9 business domains aligned with organizational ownership:

| Domain | Primary Owner | Bronze Inputs | Est. Tables | Pod Assignment |
|---|---|---|---|---|
| Enterprise Security Operations | Security Operations | Network, Windows, Audit, SaaS | 8 | Pod 1 |
| Enterprise Infrastructure Operations | Infrastructure Engineering | VMware, Windows, K8s Platform | 8 | Pod 1 |
| Enterprise Cloud Operations | Cloud Platform Engineering | Azure, AWS, GCP | 7 | Pod 2 |
| Pharmacy Operations | Pharmacy Technology | Rx Bronze Products | 9 | Pod 2 |
| Digital Customer Experience | Digital Engineering | CRM, ROCM, Web | 7 | Pod 3 |
| Clinical Operations | Clinical Engineering | Enterprise Health | 7 | Pod 3 |
| Enterprise Integration Services | Integration Engineering | MQ, Kafka, EventHub | 7 | Pod 4 |
| Enterprise Shared Services | Enterprise Architecture | Reference Data | 8 | Pod 4 |
| Enterprise Governance & Compliance | Data Governance | Audit, Security | 7 | Shared |

**Pod model:** 4 joint KPMG/SciCom pods, each owning ~2 domains (8 Silver domains ÷ 4 pods = 2 domains/pod). Enterprise Governance & Compliance is a shared responsibility across pods.

### 4.2 Bronze-to-Silver Transformation Matrix

Each Bronze Data Product fans out to one or more Silver domains:

| Bronze Data Product | Silver Domain(s) |
|---|---|
| Cloud Platform Telemetry | Cloud Operations, Governance |
| Network Security | Security Operations |
| Windows Events | Security, Infrastructure |
| VMware Infrastructure | Infrastructure Operations |
| Kubernetes Applications | Customer Experience, Clinical, Platform |
| Kubernetes Platform | Infrastructure Operations |
| Rx Platform | Pharmacy Operations |
| Enterprise Health | Clinical Operations |
| Integration Platform | Enterprise Integration |
| Enterprise Audit | Governance & Compliance |
| Enterprise Applications | Shared Services |

This one-to-many relationship is a defining architectural characteristic — one Bronze table can produce multiple Silver tables with different normalization depths and business contexts.

### 4.3 Silver Data Quality Framework

Every Silver table enforces quality rules implemented as DLT expectations:

| Quality Rule | Implementation | Failure Action |
|---|---|---|
| **Schema Conformance** | Required attributes exist with correct types | Quarantine to error table |
| **Data Type Validation** | Field types match contract specification | Cast or quarantine |
| **Duplicate Elimination** | Dedup by event_id + timestamp window | Drop duplicate |
| **CMDB Validation** | Asset/hostname resolves against dim_asset | Flag unresolved; do not block |
| **Business Rule Validation** | Domain-specific logic (e.g., severity in range 0–5) | Flag violations; alert owner |
| **Reference Integrity** | Foreign key validation against conformed dimensions | Flag unresolved; do not block |
| **Null Threshold Monitoring** | Required fields completeness > contract threshold | Alert if threshold breached |
| **Operational Freshness** | Data arrives within SLA window | Alert if late |

Quality metrics become enterprise KPIs consumed by data governance and platform engineering teams.

---

## 5.0 Governance Implementation

### 5.1 PHI/PII Masking Pipeline

The classification boundary is: **Bronze = RESTRICTED**, **Silver = INTERNAL** (masked by DLT).

| Masking Approach | Applied To | Method |
|---|---|---|
| **Regex masking** | Known PHI/PII patterns (SSN, DOB, MRN, email) | DLT column expression with regex replacement |
| **NER masking** | Unstructured text fields containing embedded PHI | AI-assisted Named Entity Recognition in DLT pipeline |
| **Tokenization** | Fields requiring reversible masking for authorized users | Token vault with Unity Catalog column-level access |
| **Column-level tags** | All Silver tables | Unity Catalog tags (PHI, PII, PCI, INTERNAL, RESTRICTED) |
| **Row-level security** | Multi-owner Silver tables | Unity Catalog row filters based on business_domain column |

CVS must approve masking rules, classification decisions, and access policies before implementation (SOW assumption).

### 5.2 Unity Catalog Governance Configuration

| Governance Element | Implementation |
|---|---|
| **Catalog structure** | One catalog per data domain (~8–12 catalogs); schemas: bronze, silver, gold |
| **Table ownership** | Mapped to B1 data contract owner; registered as Unity Catalog table property |
| **Column tags** | PHI, PII, PCI, data_classification, sensitivity_level |
| **Row filters** | Dynamic row-level security for multi-owner tables (filter by business_domain) |
| **Column masking** | Applied to tagged columns; authorized roles see unmasked values |
| **Lineage** | Automatically tracked by DLT; visible in Unity Catalog lineage graph |
| **Audit trail** | All access logged for HIPAA compliance evidence |
| **Data quality** | DLT expectations surfaced as Unity Catalog quality metrics |

### 5.3 HIPAA Compliance Evidence

WS-C builds the audit trail that supports HIPAA compliance:

- **Access audit:** Unity Catalog logs every query against PHI-containing tables
- **Masking evidence:** DLT pipeline logs confirming PHI was masked before Silver promotion
- **Classification audit:** Unity Catalog tags proving data classification at every layer
- **Lineage evidence:** End-to-end lineage from Bronze (RESTRICTED) through masking to Silver (INTERNAL)
- **Retention evidence:** Delta Time Travel + lifecycle policies proving retention compliance

---

## 6.0 Data Product Promotion

### 6.1 What Makes a Silver Table a "Data Product"

Not every Silver table is a Data Product. Data Products are Silver tables with known downstream consumers that have been formally promoted with additional metadata and governance:

| Attribute | Regular Silver Table | Silver Data Product |
|---|---|---|
| Schema governed | Yes | Yes |
| Quality expectations | Yes | Yes + published SLOs |
| Lineage tracked | Yes | Yes |
| **Purpose documented** | No | Yes — explicit business purpose |
| **Owner registered** | Yes (from B1 contract) | Yes + product owner (business stakeholder) |
| **Consumers documented** | No | Yes — named consumer teams and use cases |
| **SLO published** | No | Yes — freshness, completeness, availability targets |
| **Refresh cadence committed** | No | Yes — committed schedule |
| **Discoverable in catalog** | Yes (searchable) | Yes + promoted with documentation |

### 6.2 Promotion Criteria

A Silver table is promoted to Data Product status when:

1. At least one downstream consumer (Gold product, dashboard, report, AI model) actively depends on it
2. A product owner (business stakeholder, not just technical owner) is identified and agrees to own it
3. SLOs are defined and committed (freshness, completeness, availability)
4. Documentation is published in Unity Catalog (purpose, schema description, usage examples)
5. Quality expectations are passing at >99% rate

Tables that don't meet promotion criteria remain as governed Silver tables — retained, searchable, queryable, but not marketed as official enterprise data products.

---

## 7.0 Gold Layer and Self-Service Analytics

### 7.1 Gold Product Architecture

12 Enterprise Gold Business Products aggregate and model Silver data around business outcomes:

| Gold Product | Primary Consumer | Supporting Silver Domains | Key Outputs |
|---|---|---|---|
| Executive Enterprise Operations | CIO, CTO, Leadership | All | Availability, MTTR, SLA compliance, operational score |
| Enterprise Security Intelligence | SOC, CISO | Security Operations | Threat summary, attack surface, endpoint health, identity risk |
| Infrastructure Reliability | Infrastructure Ops | Infrastructure | Host health, capacity, availability, resource utilization |
| Cloud Operations & FinOps | Cloud Engineering | Cloud Operations | Service health, compute/storage/network summary, cost metrics |
| Pharmacy Operations Intelligence | Pharmacy Leadership | Pharmacy | Daily operations, transaction summary, platform health, business KPIs |
| Digital Customer Experience | Digital Business | Customer Experience | Session metrics, API health, journey analytics, performance |
| Clinical Operations Intelligence | Clinical Leadership | Clinical | Platform health, transaction summary, service availability |
| Enterprise Integration Intelligence | Middleware Engineering | Integration Services | Message flow, API summary, queue metrics, platform health |
| Enterprise Governance & Compliance | Audit & Compliance | Governance | Compliance dashboard, audit evidence, lineage summary, risk indicators |
| Enterprise Asset Intelligence | Enterprise Architecture | Shared Services | Asset inventory, application inventory, CMDB validation |
| AI Observability Platform | Data Science | All | Root cause features, anomaly detection, capacity prediction, incident prediction, ML feature store |
| Databricks Genie Semantic Layer | Enterprise Users | All | Natural language query interface across all domains |

### 7.2 Gold Workspace Governance

Team-owned Gold workspaces provide self-service analytics with guardrails:

| Governance Element | Implementation |
|---|---|
| **Compute governance** | Per-workspace compute budgets; auto-termination policies; serverless SQL for ad-hoc |
| **Chargeback alignment** | Compute costs attributed to consuming business unit via Unity Catalog workspace tags |
| **Access control** | Gold workspace users can read Silver (governed); can create/modify Gold tables in their workspace |
| **Promotion path** | Gold content that proves broadly useful can be promoted back to Silver via quality gate |
| **Guardrails** | No direct Bronze access from Gold workspaces; Silver is the governed access layer |

### 7.3 Genie Semantic Layer Configuration

Databricks Genie enables natural language queries against Gold semantic models:

| Genie Semantic Model | Business Domain | Example Queries |
|---|---|---|
| `genie.executive_operations` | Executive | "What is our overall platform availability this week?" |
| `genie.security_operations` | Security | "Show all Sev-1 incidents impacting customer-facing services" |
| `genie.infrastructure` | Infrastructure | "Which Kubernetes clusters are trending toward resource exhaustion?" |
| `genie.cloud_operations` | Cloud | "Which cloud services generated the highest operational cost this month?" |
| `genie.pharmacy` | Pharmacy | "Which pharmacy applications experienced the highest latency yesterday?" |
| `genie.customer_experience` | Digital | "What is the API error rate for customer-facing services?" |
| `genie.clinical_operations` | Clinical | "Show clinical platform availability for the past 30 days" |
| `genie.enterprise_assets` | Assets | "How many infrastructure assets are approaching end-of-life?" |
| `genie.finops` | FinOps | "What is our month-over-month cloud spend trend by business unit?" |
| `genie.executive_scorecard` | Executive | "Give me the executive operational scorecard for this quarter" |

Each Genie model is configured with: table references (Gold tables), field descriptions, business term glossary, sample questions, and access policies.

### 7.4 Starter Content for Gold Workspaces

To accelerate user adoption (coordinated with WS-E), WS-C provides initial content in each Gold workspace:

- **Priority SPL → SQL conversions** (Blaze-assisted) from WS-D migration backlog
- **Dashboard templates** built against Silver/Gold schemas
- **Notebook templates** for common analytical patterns (time-series trending, anomaly detection, capacity forecasting)
- **SQL query library** with documented examples per domain
- **Best practices guide** for Databricks SQL, Notebooks, and Genie usage

---

## 8.0 Delivery Cadence

### 8.1 Build Phase (Weeks 1–10)

| Period | Activities | Output |
|---|---|---|
| Weeks 1–3 | Canonical data model design; conformed dimension specification; DLT template architecture; governance framework design | Canonical model doc; dimension specs; template design; governance design |
| Weeks 4–6 | Build first 4–6 DLT templates (Security OCSF, Infrastructure, Cloud); implement governance patterns (masking, RLS, tags); configure first 2 Silver domains | Template library v1; 2 Silver domains operational; governance patterns proven |
| Weeks 7–10 | Scale to remaining template categories; instantiate pipelines for Tier 1 source families; configure Gold workspace structure; Genie semantic model setup | Template library v2; Tier 1 Silver tables live; Gold workspaces configured |

### 8.2 Operate Phase (Weeks 11–24)

| Period | Activities | Output |
|---|---|---|
| Weeks 11–14 | Instantiate remaining Silver domains (Pods 2–4); onboard Tier 2 source families; data product promotion for highest-value tables | 6–8 Silver domains live; data products promoted |
| Weeks 15–18 | Complete Silver domain coverage; build Gold analytical products; configure remaining Genie models; enrichment integrations (CMDB, ServiceNow) | All 9 Silver domains live; Gold products available |
| Weeks 19–22 | Gold starter content deployment; AI feature store configuration; pipeline optimization and tuning | Gold workspaces with content; AI platform ready |
| Weeks 23–24 | Documentation finalization; operational runbooks; BAU handoff preparation; knowledge transfer to CVS | Production-ready platform; handoff materials |

### 8.3 Pod Velocity Metrics

| Metric | Build Phase Target | Operate Phase Target |
|---|---|---|
| DLT templates created per sprint | 2–3 | 0–1 (maintenance) |
| Silver tables instantiated per sprint per pod | 3–5 | 8–12 |
| Data products promoted per sprint | 1–2 | 3–5 |
| Enrichment integrations completed per sprint | 0–1 | 1–2 |
| Gold products delivered per sprint | 0 | 1–2 |

---

## 9.0 Alignment with SOW Scope and Deliverables

| SOW Deliverable | Methodology Section |
|---|---|
| Data Architecture and Pipeline Design | Sections 2 (Canonical Model) + 3 (Templates) + 4 (Silver Domains) |
| Reusable DLT/PySpark Template Library and Documentation | Section 3 (Template Engineering) |
| Bronze-to-Silver Pipeline Inventory and Implementation Evidence | Section 4 (Silver Domain Implementation) |
| Canonical Data Model and Source-to-Model-to-Ontology Mapping Strategy | Section 2 (Canonical Model and Ontology) |
| Silver Data Model and Data Product Catalog | Sections 4 (Domains) + 6 (Data Product Promotion) |
| Conformed Dimensions and Enterprise Semantic Model Documentation | Section 2.4 (Conformed Dimensions) |
| Unity Catalog Governance Configuration Documentation | Section 5 (Governance Implementation) |
| Enrichment Integration Design and Validation Evidence | Section 4.2 (Transformation Matrix) + Section 8 (enrichment in Operate phase) |
| Gold Workspace/Self-Service Analytics Design | Section 7 (Gold Layer and Self-Service) |
| Pipeline Monitoring, SLO and Operational Runbooks | Sections 4.3 (Quality Framework) + 6 (SLO publication) + 8 (Operate phase runbooks) |

---

## 10.0 Coordination with Other Workstreams

| Workstream | Dependency | Coordination Point |
|---|---|---|
| **B1 → C** | B1 data contracts define pipeline input specifications (schema, SLA, classification, masking rules) | C cannot build pipelines without approved B1 contracts for target source families |
| **B2 → C** | B2 provides populated Bronze tables | C pipeline development begins once B2 confirms Tier 1 Bronze tables have production data |
| **C → D** | C delivers Silver/Gold tables that D's migrated queries target | D cannot validate converted SPL until C's Silver tables contain queryable data |
| **C → E** | C configures Gold workspaces that E's user migration waves provision users into | E adoption waves sequenced to match C Gold workspace readiness |
| **A → C** | A validates architecture and produces pilot implementation | C inherits A's pilot patterns and remediation findings as design inputs |
| **C → F** | C produces operational runbooks and monitoring baselines | F inherits C's pipeline monitoring during hypercare |
