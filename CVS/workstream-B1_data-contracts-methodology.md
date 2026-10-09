# Workstream B1: Data Contract Collection and Rationalization — Detailed Methodology

## 1.0 Approach Overview

This document describes the detailed methodology KPMG will employ to advise on, accelerate, and support data contract collection and rationalization for CVS Health's Enterprise Observability and Security Data Foundation program. It supplements SOW Section 2.4 (Workstream B1) by providing the specific process, engagement model, decision framework, and acceleration techniques that KPMG's embedded resources bring to CVS's in-flight data contracts effort.

**The Problem:**  
The CVS Splunk environment contains 12,910 distinct sourcetypes generating approximately 207 billion events per day. However, analysis demonstrates that the vast majority (>80%) are implementation artifacts — Kubernetes deployment labels, daemon process names, environment-specific naming conventions — not unique data schemas. This creates massive redundancy, ungoverned sprawl, and blocks AI readiness.

**The Objective:**  
Rationalize 12,910 raw sourcetypes into approximately 250–300 logical source families governed by Open Data Contract Standard (ODCS) contracts. Each contract defines ownership, canonical schema, security classification, data quality SLAs, and lineage — creating the enforceable foundation upon which Bronze ingestion, Silver transformation, Gold analytics, and AI enablement depend.

**Relationship to Other Workstreams:**
- Workstream A validates the platform architecture that contracts will govern
- Workstream B2 consumes contracts to configure Cribl routing and Bronze landing
- Workstream C consumes contracts to build Silver/Gold pipelines
- Workstream D consumes contracts to understand which analytics content maps to which data products
- Workstream E communicates contract-driven changes to user communities

**Engagement Scope Boundary:**
CVS's data contract effort is already in flight under Chuck Marcum and Tim Janikowski's team, targeting ~14,000 legacy data sets that collapse to ~2,000–2,500 populated contracts via parent-child relationships (Jul 31 session). KPMG's rationalization methodology further reduces this to ~250–300 logical source families through domain/family grouping (Section 2). KPMG does not own completion of the data contracts; CVS retains ownership of the attestation and registration process. KPMG embeds 1–3 business-analyst resources to formalize, structure, and align the process to best practices — providing the business-unit and domain/technology perspective that complements Chuck's data-centric top-talker analysis. Each contract may govern dozens or hundreds of underlying sourcetypes (e.g., a single "Network Security" contract covers Cisco, PAN, F5, Check Point, Fortinet, and Infoblox sourcetypes). The 250–300 source family taxonomy is validated during the pre-assessment discovery phase. If discovery reveals materially different family counts, scope adjustment is handled via SOW Section 9 (Change Order).

**Scope Rationale — Ownership and Organizational Reality (Aug 3 alignment):**
Of the ~230–240 core Splunk users mapping to 10–11 lines of business, only one business unit sits under the project sponsor's (Alan Rose's) direct control. Taking ownership or liability for contract completion across all lines of business would require organizational authority KPMG does not have. The embedded-analyst model keeps KPMG accountable for methodology quality, domain mapping, and acceleration — while CVS retains accountability for securing data-owner attestation across business units.

**Critical Path Status:**
Data contracts are not on the critical path for the migration itself. Rebuilt queries in Workstream D will independently inform the data types and fields needed for PySpark/SQL translation. However, contracts remain essential for data longevity, retention policy, regulatory compliance, and post-migration governance — largely CVS-owned concerns that KPMG's methodology formalizes.

**Critical Dependency — Data-Owner Interaction:**
Data contracts require validation from actual source and data-derivative owners across CVS — not solely the observability team KPMG interfaces with daily. Effective coverage of these interactions (per Josh's direction) requires structured engagement with application owners, infrastructure teams, and data stewards across the enterprise. The methodology accounts for this through the wave-based, pre-populated engagement model described in Section 4, but delivery velocity remains gated by CVS's ability to secure data-owner time.

---

## 2.0 Source Rationalization Methodology

### 2.1 The Rationalization Funnel

Source rationalization follows a four-step structured funnel. Each step reduces complexity while increasing governance and business value:

```
Step 1: Automated Discovery & Grouping
    12,910 raw sourcetypes → ~250–300 logical source families (50:1 ratio)

Step 2: Classification & Scoring
    Score each family by business value, usage, duplication, complexity, sensitivity
    → Assign Tier 1 / Tier 2 / Tier 3 priority

Step 3: Disposition Decision
    For each family: Retain & Contract / Consolidate / Retire

Step 4: Contract Authoring
    Generate ODCS contract for each retained or consolidated family
    → Register in Unity Catalog
```

### 2.2 Step 1 — Automated Discovery & Grouping

**Objective:** Collapse 12,910 raw sourcetypes into ~250–300 logical source families using pattern recognition and domain knowledge.

**Method:**
1. Export Splunk source type inventory (names, indexes, event counts, last-seen timestamps, schema fingerprints)
2. Apply AI-assisted pattern recognition to identify families:
   - **Naming pattern analysis:** Group sourcetypes sharing common prefixes/suffixes (e.g., `kube:container:crm-api-prod`, `kube:container:crm-api-qa` → single "CRM" family)
   - **Schema similarity:** Cluster sourcetypes with identical or near-identical field structures
   - **Index co-location:** Group sourcetypes sharing the same Splunk index
   - **Metadata inheritance:** Identify sourcetypes that share app ownership, tags, or event types
3. Map resulting families into 12 enterprise domain categories:

| Domain Category | Est. Families | Representative Collapse |
|---|---|---|
| K8s Application Platform | ~20 | 10,790 K8s sourcetypes → ~40 logical families |
| Network & Security | 40–60 | Cisco, PAN, F5, Checkpoint, Fortinet, Infoblox |
| VMware Infrastructure | 10–20 | 307 raw ESXi/vCenter sourcetypes → few schemas |
| Windows Platform | 15–20 | WinEventLog, AD, DNS, DHCP, PowerShell, Sysmon |
| Linux/Unix | ~15 | Syslog, Auditd, SSH, Kernel, Systemd |
| Web & Middleware | ~20 | Apache, IIS, Tomcat, WebSphere, DataPower |
| Cloud Providers | ~20 | Azure, AWS, GCP activity/audit/security logs |
| SaaS Platforms | ~20 | O365, CrowdStrike, Mimecast, Okta, Zscaler |
| Databases | ~20 | Oracle, SQL Server, DB2, PostgreSQL, MongoDB |
| Enterprise Applications | 60–80 | Pharmacy, Claims, Aetna, PBM, Loyalty, Retail |
| Messaging & Streaming | ~10 | Kafka, MQ, Event Hub, Pub/Sub |
| Observability & Monitoring | ~20 | Prometheus, Grafana, Datadog, OpenTelemetry |

**Kubernetes example:** The single largest rationalization opportunity. 10,790 sourcetypes (84% of the total inventory) are Kubernetes deployment-specific names encoding application, environment, and deployment metadata directly into the sourcetype string. These collapse into approximately 40 logical application families where deployment, namespace, cluster, and environment become columns in a single Bronze table rather than separate schemas. CVS confirmed (Jul 31) that **over 10,000 data contracts** are tied to Kubernetes due to Splunk's dynamic source typing — every new service deployed gets its own data set. Chuck Marcum gave the example of "Cloud Engineering AKS," which shows ~5,000 source types today but really emits only ~4 distinct source types. Kubernetes and VMware are the priority consolidation targets ("biggest trees to chop"), with VMware segregated down to the DaemonSet level to remove a further several thousand contracts.

**Output:** Source Family Taxonomy — a catalog mapping every raw sourcetype to its logical family and domain category.

### 2.3 Step 2 — Classification & Scoring

**Objective:** Prioritize source families for contract engagement based on business value and migration readiness.

**Scoring Criteria (per family):**

| Criterion | Weight | Measurement |
|---|---|---|
| Business Value | High | Supports active operational/security decisions? |
| Active Usage | High | Events observed in last 30 days? Active consumers? |
| Data Volume | Medium | Daily TB contribution (cost reduction opportunity) |
| Downstream Dependencies | Medium | Feeds active searches, alerts, dashboards? |
| Duplication/Overlap | Medium | Multiple families serving same business purpose? |
| Security Sensitivity | Medium | Contains PHI, PII, or PCI-regulated data? |
| Migration Complexity | Low | Schema stability, transformation difficulty |

**Tier Assignment:**

| Tier | Criteria | Contract Priority | Estimated Count |
|---|---|---|---|
| Tier 1 | High business value + active usage + high volume | Weeks 1–10 | ~75–100 families |
| Tier 2 | Moderate value or moderate usage | Weeks 8–16 | ~100–125 families |
| Tier 3 | Low volume, limited usage, long-tail | Weeks 14–20 | ~50–75 families |

### 2.3.1 Cross-Platform Source Inventory

Classification must account for the full data landscape — not only Splunk Core. Source families are drawn from three platform domains, each contributing distinct data characteristics:

| Platform Domain | Contribution to Source Inventory | Contract Implications |
|---|---|---|
| **Splunk Core** | Majority of the 12,910 sourcetypes; operational and application telemetry | Standard contract flow; highest volume |
| **Enterprise Security (ES)** | Security-specific sources (correlation inputs, threat feeds, compliance logs) | Higher security classification; stricter SLAs; regulatory constraints |
| **AI Ops / ITSI** | Service health inputs, KPI data sources, anomaly detection feeds | ML model dependencies; real-time freshness requirements |

Some source families span multiple platform domains (e.g., network firewall logs consumed by both Core operational dashboards and ES correlation searches). When a source family serves multiple domains:
- The contract reflects the **highest classification** requirement across all consumers
- SLA targets are set by the **most demanding** downstream consumer
- The family is tagged for **cross-domain validation** during owner review

This ensures the rationalization inventory represents the complete data universe and that contracts are authored once per family, regardless of how many platform domains consume it.

### 2.3.2 Source Family Count Reconciliation

Two analyses establish the source family landscape at different scope levels:

- **Source Type Rationalization (Jul 12, 2026):** Analyzed the full 12,910 sourcetype inventory across all 12 domain categories and estimated **250–300 logical source families** at a ~50:1 rationalization ratio. This is the complete taxonomy used for contract scoping.
- **Technical Implementation Approach (Jul 26, 2026):** Validated the **top 2,000 production sourcetypes** (by 24-hour volume) and identified **126 physical source families** accounting for **145.4 TB/day** (~98% of daily telemetry volume). This is the confirmed production core.

These are not in conflict — they represent different scope boundaries within the same rationalization funnel:

| Scope | Family Count | Coverage | Contract Priority |
|---|---|---|---|
| **production-validated (7/26)** | ~126 families | Top 2,000 sourcetypes; 145.4 TB/day; ~98% of volume | **Tier 1** — contracted first (Weeks 1–10) |
| **full taxonomy (7/12)** | 250–300 families | All 12,910 sourcetypes including long-tail | Tier 1 + Tier 2 + Tier 3 over 20 weeks |
| **Delta (long-tail)** | ~124–174 families | Remaining sourcetypes; <2% of volume | **Tier 2/3** — lower volume but potential business criticality |

The 126 production-validated families form the natural Tier 1 priority set. They represent the sources that consume the most infrastructure, drive the most analytics, and provide the greatest cost-reduction opportunity when migrated to Databricks. The remaining 124–174 families represent lower-volume, potentially niche sources that may still carry business criticality (compliance, audit, seasonal reporting) and require owner validation before disposition.

**Kubernetes Dominance — The Biggest Single Rationalization Factor:**
The K8s Rationalization Analysis (Jul 12, 2026) demonstrates that **10,790 Kubernetes sourcetypes (84% of the entire Splunk inventory)** collapse into **~40 logical application platforms**. Kubernetes deployment names encode application, environment, and deployment metadata directly into the sourcetype string — these are operational metadata, not unique schemas. The rationalization from 10,790 → ~40 is achieved by promoting deployment, namespace, cluster, and environment into columns within Bronze tables rather than treating them as separate schemas. This single-domain rationalization accounts for the majority of the 50:1 compression ratio across the full inventory.

| K8s Application Family | Representative Sourcetypes | Bronze Target |
|---|---|---|
| CRM Applications | `kube:container:crm-*` | `bronze_crm` |
| Pharmacy Platform | `kube:container:rxdw-*` | `bronze_rxdw` |
| Pharmacy AI | `kube:container:rphai-*` | `bronze_rphai` |
| Customer Opportunity | `kube:container:rocm-*` | `bronze_rocm` |
| Care Engine | `kube:container:cee-*` | `bronze_cee` |
| Aetna Platform | `kube:container:aetna-*` | `bronze_aetna` |
| K8s Platform Services | Istio, Envoy, Gatekeeper, ArgoCD, `kube-*` | `bronze_k8s_platform` |

Both analyses converge on **~25 Bronze ingestion tables** — demonstrating that even at the family level, significant further consolidation occurs at the physical storage layer (multiple families → single Bronze table per domain).

### 2.4 Step 3 — Disposition Decision

**Objective:** Determine what happens to each source family — not everything migrates.

Every source family receives one of three dispositions:

**Retain & Contract (target: ~200–250 families)**
- Active operational or business value confirmed
- Source owner identified and engaged
- ODCS contract authored and registered
- Proceeds to Bronze ingestion → Silver transformation

**Consolidate (target: ~30–50 families merged into retained families)**
- Duplicates or overlaps with another family serving the same business purpose
- Merged into the primary family's contract (schema unified, ownership consolidated)
- Reduces total contract count while preserving data coverage

**Retire (target: ~20–40 families)**
- No active consumers, no events in 90+ days, no downstream dependencies
- Owner confirms retirement (or no owner can be identified after escalation)
- Cataloged, archived, excluded from migration scope
- Aligns with the broader rationalization of 12,667 disabled saved searches and 983 inactive dashboards

**Analytics Content Rationalization (feeds disposition):**

Source disposition decisions are informed by a parallel rationalization of the analytics content that depends on each source family. Content is classified using a four-category framework:

| Category | Description | Count | Action |
|---|---|---|---|
| A – Active Scheduled Searches | Enabled scheduled searches generating alerts, ServiceNow incidents, reports, compliance output | 15,159 | Primary migration candidates — convert to Databricks Jobs/Workflows (WS-D) |
| B – Historical Analytics | Searches routinely querying data beyond the planned Splunk hot-retention window (weekly, monthly, quarterly reports, capacity planning) | TBD | Migrate early — Databricks becomes authoritative platform for long-term telemetry |
| C – High-Cost Scheduler Workloads | Searches with high execution frequency, long runtime, or significant data scan volumes | TBD | Prioritize for immediate Splunk licensing and infrastructure cost relief |
| D – Disabled Searches (Retire) | Disabled or unused searches | 12,667 (38%) | Archive, catalog, approve for retirement — do not migrate or contract |

**Dashboard Rationalization (feeds disposition):**

| Dashboard Status | Count | % | Disposition |
|---|---|---|---|
| Accessed within last 7 days | 1,295 | 38% | Highest operational priority — evaluate for Databricks migration |
| Accessed within last 30 days | 2,430 | 71% | Retain or migrate based on business need |
| Not accessed in 30+ days | 983 | 29% | Business review required before migration; retirement candidates |

Dashboards not accessed in 30+ days are not automatically retired. Each undergoes business validation considering: owner confirmation, user access history, dependencies, supporting applications, regulatory/audit requirements, and replacement by newer dashboards. Only dashboards that complete this review enter the migration path.

**Knowledge Object Rationalization (feeds disposition and contracts):**

Supporting Splunk knowledge objects are assessed alongside their owning applications and mapped to Databricks equivalents:

| Object Type | Count | Rationalization Strategy | Databricks Equivalent |
|---|---|---|---|
| Lookups | >1,000 | Migrate with owning application | Unity Catalog managed tables / Delta tables |
| Macros | 587 | Rewrite where required | SQL UDFs / Notebook functions |
| Event Types | TBD | Evaluate individually | Silver-layer filter logic |
| Tags | TBD | Preserve as metadata | Unity Catalog tags |
| Calculated Fields | TBD | Implement within Silver transformations | Silver DLT column expressions |
| Data Models | 45 | Re-engineer using Medallion architecture | Silver domain tables + Gold products |
| KV Store Collections | 201 | Validate application dependency | Delta tables with merge/upsert patterns |

Knowledge objects that are only referenced by retired searches or dashboards become retirement candidates themselves. Objects shared across multiple active searches are flagged as high-priority migration dependencies.

**Application-Level Rationalization:**

The 552 Splunk applications will be rationalized on an application-by-application basis. Each application receives a migration assessment documenting: associated indexes, source families, scheduled searches, dashboards, lookups, macros, KV Store dependencies, user communities, business criticality, and recommended migration wave. Applications with no active users or content are retirement candidates. Applications whose content has been fully migrated to Databricks are candidates for decommissioning.

**How disposition feeds downstream workstreams:**

If all downstream consumers (searches, dashboards, knowledge objects) of a source family are themselves being retired, the source family becomes a retirement candidate — it no longer needs a data contract. Conversely, source families with active Category A or B consumers are confirmed Retain candidates and proceed to contract authoring.

**Disposition Decision Framework:**

```
                         ┌─────────────────────┐
                         │  Source Family       │
                         │  (from Step 2)       │
                         └──────────┬───────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            Active Usage?     Duplicate of      No events 90+ days
            Owner exists?     another family?   No owner found?
                    │               │               │
                    ▼               ▼               ▼
            ┌───────────┐   ┌──────────────┐  ┌─────────┐
            │  RETAIN   │   │ CONSOLIDATE  │  │ RETIRE  │
            │ & CONTRACT│   │ (merge into  │  │         │
            └───────────┘   │  primary)    │  └─────────┘
                            └──────────────┘
```

### 2.5 Step 4 — Contract Authoring

**Objective:** Produce a governed ODCS contract for every retained or consolidated source family.

Once disposition is confirmed and source owner is engaged, KPMG generates the formal data contract. This process is heavily AI-assisted (see Section 5.0) with mandatory human validation.

**Authoring Workflow:**
1. AI pre-populates contract template from source metadata (schema, volume, index, owner directory)
2. KPMG engineer reviews and completes security classification, quality rules, and lineage
3. Source owner workshop: validate schema, confirm ownership, approve SLAs
4. KPMG finalizes contract and registers in Unity Catalog
5. Contract becomes gate for downstream workstreams (B2 ingestion, C pipeline, D analytics)

---

## 3.0 Consolidated Source Contract Structure

Each ODCS contract represents a **consolidated governance agreement** for a logical source family — potentially covering dozens or hundreds of underlying Splunk sourcetypes under a single governed contract.

### 3.1 Contract Sections

| Section | Purpose |
|---|---|
| **Ownership & Accountability** | Names the data owner, steward, business domain, and approval authority |
| **Schema Definition** | Defines canonical fields, data types, required vs. optional, schema version |
| **Security & Compliance** | Declares PHI/PII/PCI classification, masking requirements, access policies, workspace isolation |
| **Data Quality & SLAs** | Codifies freshness target, completeness threshold, availability, violation alerting |
| **Lineage & Interfaces** | Maps upstream sources, downstream Silver consumers, transformation contracts, enrichment dependencies |

### 3.2 Representative Contract Example

```yaml
contract:
  id: bronze_network_security_v1
  version: "1.0"
  status: active
  effective_date: "2026-10-01"

ownership:
  data_owner: "Network Security Engineering"
  data_steward: "J. Smith"
  business_domain: "Infrastructure & Security"
  approval_authority: "Chris Robinson"
  contact: "network-security-data@cvshealth.com"

source_families:
  - family: "Cisco ASA"
    raw_sourcetypes: ["cisco:asa", "cisco:asa:*"]
    splunk_indexes: ["network", "firewall"]
  - family: "Palo Alto"
    raw_sourcetypes: ["pan:traffic", "pan:threat", "pan:system"]
    splunk_indexes: ["pan_logs"]
  - family: "F5"
    raw_sourcetypes: ["f5:bigip:*"]
    splunk_indexes: ["f5"]
  - family: "Check Point"
    raw_sourcetypes: ["checkpoint:*"]
    splunk_indexes: ["checkpoint"]
  - family: "Fortinet"
    raw_sourcetypes: ["fortinet:*"]
    splunk_indexes: ["fortinet"]
  - family: "Infoblox"
    raw_sourcetypes: ["infoblox:*"]
    splunk_indexes: ["infoblox"]

bronze_target:
  table: "bronze.network_syslog"
  partition_key: ["event_date", "vendor"]
  format: "Delta"

schema:
  version: "1.0"
  required_fields:
    - name: "event_timestamp"
      type: "timestamp"
      description: "Original event generation time"
    - name: "vendor"
      type: "string"
      description: "Source vendor (Cisco, PAN, F5, etc.)"
    - name: "hostname"
      type: "string"
      description: "Device hostname"
    - name: "action"
      type: "string"
      description: "Allow, Deny, Reset, Drop"
    - name: "severity"
      type: "integer"
      description: "Normalized severity (0-5)"
    - name: "raw_payload"
      type: "string"
      description: "Original event preserved without modification"
  optional_fields:
    - name: "src_ip"
      type: "string"
    - name: "dst_ip"
      type: "string"
    - name: "src_port"
      type: "integer"
    - name: "dst_port"
      type: "integer"
    - name: "protocol"
      type: "string"

security:
  classification: "INTERNAL"
  contains_phi: false
  contains_pii: true  # IP addresses
  masking_rules:
    - field: "src_ip"
      rule: "mask_in_gold_external"
    - field: "dst_ip"
      rule: "mask_in_gold_external"
  pii_masking_layer: "cribl_guard_pre_ingestion"  # Confirmed Jul 31: PII/PHI masked at Cribl Guard before data traverses the wire
  workspace_isolation: "security_workspace"
  row_level_security: "by business_domain"

quality:
  freshness_sla: "5 minutes"
  completeness_threshold: "99.5%"
  availability_target: "99.9%"
  null_tolerance:
    event_timestamp: "0%"
    vendor: "0%"
    hostname: "5%"
  violation_workflow:
    alert: "Unity Catalog data quality alert"
    escalation: "ServiceNow incident after 3 consecutive failures"

lineage:
  upstream:
    - source: "Cribl Stream"
      pathway: "/bronze/security/network/"
      format: "JSON"
  downstream:
    - target: "silver.security_network_sessions"
      transformation: "Normalize vendor schemas, enrich threat intel, deduplicate"
    - target: "silver.security_firewall_events"
      transformation: "Parse action/rule fields, MITRE mapping"
  enrichment_dependencies:
    - "dim_asset (CMDB hostname resolution)"
    - "threat_intel_feed (IP reputation)"

retention:
  bronze: "13 months"
  silver: "13 months"
  cold_archive: "7 years (WORM)"
```

This single contract governs six vendor families (Cisco, PAN, F5, Check Point, Fortinet, Infoblox) and potentially hundreds of underlying raw sourcetypes — demonstrating the "consolidated source contract" model Josh specified.

---

## 4.0 Source-Owner Engagement Model

### 4.1 Engagement Principles

KPMG embeds 1–3 business-analyst resources into CVS's existing data contracts team to formalize, structure, and accelerate the process — not to own contract completion. The engagement model complements CVS's data-centric approach (Chuck's top-talker/user analysis) with KPMG's business-unit and domain/technology perspective:

- **Embedded support, not ownership:** KPMG analysts work alongside Chuck, Tim, and Annette's team — aligning contracts to domain families and bridging to Databricks ingestion requirements
- **Wave-based, not all-at-once:** Tier 1 families engage first; later tiers benefit from established templates and patterns
- **Pre-populated, not blank-slate:** AI generates draft contracts before owner engagement — owners validate, not author from scratch; CVS's auto-populated field lists and "important" field designations are the starting point
- **Domain-grouped workshops:** Batch related source owners (e.g., all Network Security owners in one session) to reduce scheduling overhead; align to the ~10 LOBs identified by SciComm
- **Time-boxed review cycles:** Owners have defined review windows (5 business days) with governance escalation for non-response

**Critical Dependency — Data-Owner Availability:**
KPMG's contract authoring capacity (15–20 contracts per sprint) is not the binding constraint on delivery velocity. The binding constraint is CVS securing data-owner availability. Data contracts require validation and sign-off from application and infrastructure teams — the actual data owners and stewards — who are not the observability team KPMG works with day-to-day. KPMG can pre-populate contracts via AI and have them ready for review, but owner validation is gated by CVS's ability to schedule data-owner time. If data-owner access is limited or delayed, contract completion rate will be proportionally constrained regardless of KPMG engineering capacity. The SOW assumption that "CVS source owners will respond to contract validation and approval requests within expected review windows" is the critical dependency underpinning the 250–300 contract target.

**Mitigating Factor — Ownership Concentration (validated Jul 31):**
The Jul 31 session confirmed that ownership concentration is highly favorable: the **top 25 source owners** account for **over 90%** of Splunk data, with the top 2–5 owners alone covering ~80%+. CVS plans white-glove attestation sessions with these top-25 owners (starting week of Aug 10), clustering them by line of business (~10 LOBs: retail, clinical, pharmacy, digital, ISTS, etc.) via the AD path to leadership. This means the critical mass of contract coverage can be achieved through fewer than 100 structured meetings — a tractable engagement scope. However, the enforcement gap remains: new onboarding has natural leverage (must complete a contract to get into Splunk), but legacy data owners face no penalty for non-completion since their data is already ingested. Leadership push from Josh, O'Connor, and VPs under Alan Rose is needed to compel attestation from application owners outside the direct chain.

**Ownership Complexity — Many-to-Many Reality:**
Source type ownership at CVS is many-to-many and messy. With ~3,000–4,000 Splunk users, a single source type can have multiple owners across infrastructure, network, database, and application teams — all wanting visibility into the same telemetry for different purposes. The engagement model must account for this overlap: a single data contract may require sign-off from multiple stakeholders with different analytical interests in the same source family. Domain-grouped workshops (Section 4.3) are designed to surface and resolve these overlapping ownership claims efficiently.

**Leveraging Existing CVS Assets (validated Jul 31, 2026):**
The data contracts methodology session with Chuck Marcum (technology tool owner), Tim Janikowski (Director, Observability Governance), Max Rao (Product Owner, Data Lake/Contracts), and Annette Goodman (Business Ops/Team Lead) confirmed that CVS has built substantial infrastructure KPMG can accelerate rather than rebuild:

- **Auto-populated contract inventory:** Chuck's team has parsed Splunk indexes and source types to auto-populate field names for ~14,000 legacy data sets (active in the last 60 days). Fields appearing in SPL queries executed over a 15-day window are pre-marked as "important" — these are the fields that will go through optimization in Bronze and Silver layers.
- **Ownership identification methodology:** Owners are identified programmatically via Splunk role access → Active Directory group membership → primary AD group owner (looked up in AccessNow). This has surfaced **259 legacy contract owners** so far.
- **Top-talker prioritization:** Chuck built a top-talker analysis measuring users, dashboards, events, and SVCs per index to validate actual usage and set migration priority — directly answering the question of whether data is genuinely consumed, not just ingested.
- **Two parallel workflows:** (1) A mature new-onboarding contract process (required since August 2025 for anything entering Splunk); and (2) the newly launched legacy registration effort for existing data sets. KPMG's engagement accelerates the legacy effort.
- **Parent-child contract model:** CVS is already using parent-child relationships (KPMG term: "family") to collapse similar data sets. The target is reducing ~14,000 data sets to ~2,000–2,500 populated contracts — directionally consistent with KPMG's 250–300 source family estimate after further rationalization.

KPMG confirmed strong methodological alignment with CVS's existing approach (Jul 31). The Aug 3 internal alignment further clarified the posture: KPMG will not duplicate disposition processes CVS has already built (auto-population, owner lookup, escalation app). Instead, embedded analysts focus on three value-adds: (1) providing the business-unit and domain/technology mapping that complements Chuck's data-centric analysis, (2) formalizing and structuring the process to best practices and ODCS standards, and (3) bridging the gap from contracts to Databricks ingestion and Silver-layer design.

### 4.2 Per-Source-Owner Workflow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  1. KPMG PRE-POPULATES         2. OWNER REVIEWS           3. FINALIZE   │
│  ─────────────────────         ────────────────           ───────────    │
│                                                                          │
│  • AI infers schema from       • Domain workshop          • KPMG QA      │
│    Splunk metadata               (2-hour session)           review       │
│  • Draft contract generated    • Owner confirms:          • Unity Catalog│
│  • Classification proposed       - Schema correctness       registration │
│  • SLA template applied          - Ownership accuracy     • Gate release │
│  • Lineage mapped from            - Security class.         for WS-B2/C  │
│    existing Splunk content       - SLA appropriateness                   │
│                                • Sign-off or corrections                 │
│                                                                          │
│  Timeline: 2–3 days           Timeline: 5 biz days       Timeline: 2 d  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Workshop Structure

| Session Type | Duration | Participants | Frequency |
|---|---|---|---|
| Domain Workshop | 2 hours | 3–5 source owners from same domain + KPMG data engineer | 2–3 per week |
| Schema Review | 1 hour | Individual source owner + KPMG architect | As needed |
| Escalation Review | 30 min | Domain lead + KPMG program manager | Weekly |
| Executive Checkpoint | 30 min | Josh/Chris + KPMG lead | Bi-weekly |

### 4.4 Escalation Path

| Scenario | Action | Timeline |
|---|---|---|
| Owner responds within window | Normal flow | 5 business days |
| Owner non-responsive (first miss) | KPMG follow-up + domain lead cc'd | +3 days |
| Owner non-responsive (second miss) | Escalate to program governance | +5 days |
| No owner identifiable | Flag for executive decision (retire vs. assign); treated as a **security finding** requiring resolution (per CVS policy) | Next governance review |
| Data owner outside KPMG stakeholder network | CVS program management identifies and secures the correct owner; KPMG provides pre-populated contract draft for review | CVS PM escalation; +5–10 days |
| CVS auto-escalation triggered | CVS's existing contract app sends auto-reminders and auto-escalates incomplete contracts after a set number of days; KPMG monitors and supplements with direct outreach | Automated; KPMG supplements |

### 4.5 Capacity Planning

| Metric | Target |
|---|---|
| Contracts completed per sprint (2 weeks) | 15–20 |
| Domain workshops per week | 2–3 |
| Source owners engaged per week | 8–12 |
| Total contracts over 20 weeks | 250–300 |
| Tier 1 completion target | By week 10 (Nov freeze checkpoint) |

---

## 5.0 AI-Assisted Acceleration

### 5.1 Why AI Is Required

At 15–20 contracts per sprint, the 20-week timeline requires high-efficiency execution. Manual authoring of 250–300 contracts from scratch would require significantly larger staffing. AI acceleration achieves the required throughput without linear headcount growth by automating the repetitive, pattern-recognizable portions of the work.

### 5.2 AI Capabilities Deployed

| Capability | What It Does | Human Gate |
|---|---|---|
| **Pattern Recognition** | Analyzes sourcetype naming patterns, schema fingerprints, and metadata to auto-group into families | KPMG engineer validates groupings |
| **Schema Inference** | Examines sample events and field extractions to propose canonical schema per family | Source owner confirms/corrects |
| **Quality Rule Generation** | Proposes SLA templates (freshness, completeness) based on source characteristics and domain norms | Source owner adjusts to business needs |
| **Contract Generation** | Produces complete ODCS YAML/JSON from validated inputs using parameterized templates | KPMG architect reviews before registration |
| **Lineage Mapping** | Traces existing Splunk search dependencies to identify downstream Silver consumers | KPMG engineer validates against WS-C design |
| **Classification Inference** | Proposes security classification based on index naming, field content analysis, and existing Splunk RBAC | Security SME validates |

### 5.3 Human-in-the-Loop Governance

Every AI-generated output passes through a mandatory validation gate:

- **AI output → KPMG engineer review → Source owner confirmation → Unity Catalog registration**
- No contract is registered without both KPMG quality assurance AND source owner approval
- AI suggestions are explicitly labeled as draft/proposed in all owner-facing materials
- Classification decisions (PHI/PII/PCI) always require human security SME sign-off

### 5.4 Measured Acceleration

| Activity | Without AI | With AI | Reduction |
|---|---|---|---|
| Sourcetype-to-family grouping | ~3 hours per family | ~20 min per family (review only) | ~90% |
| Schema definition per contract | ~4 hours | ~1 hour (validate, not author) | ~75% |
| Contract template population | ~2 hours | ~15 min (auto-generated) | ~87% |
| Total per-contract effort | ~12 hours | ~3–4 hours | ~70% |

This acceleration is what enables 250–300 contracts in 20 weeks with the proposed staffing model.

---

## 6.0 Delivery Cadence and Factory Model

### 6.1 Build Phase (Weeks 1–8)

**Objective:** Establish the framework, tooling, and patterns; deliver first 50 Tier 1 contracts.

| Week | Activities | Output |
|---|---|---|
| 1–2 | Export Splunk inventory; configure AI tooling; establish taxonomy framework; identify Tier 1 families | Source Family Taxonomy v1; Tier prioritization |
| 3–4 | Author contract template; pilot with 5–10 highest-value families; refine engagement workflow | Contract template finalized; 10 pilot contracts |
| 5–6 | Scale Tier 1 engagement; run first domain workshops; onboard remaining Tier 1 families | 20–30 additional contracts |
| 7–8 | Complete Tier 1; validate against B2/C requirements; Build-phase retrospective | ~50 Tier 1 contracts; lessons learned |

**Build Phase Staffing:** Higher-seniority resources (Director + Senior Engineer onshore) establishing standards, templates, and quality gates.

### 6.2 Operate Phase (Weeks 9–20)

**Objective:** Scale to full 250–300 contracts using established factory patterns with increased offshore leverage.

| Period | Activities | Output |
|---|---|---|
| Weeks 9–12 | Tier 2 engagement; domain workshops at cadence; AI handles bulk of pre-population | 60–80 additional contracts |
| Weeks 13–16 | Tier 2 completion; Tier 3 begins; consolidation decisions finalized | 60–80 additional contracts |
| Weeks 17–20 | Tier 3 completion; retirement approvals; final catalog; BAU handoff preparation | Remaining contracts; governance handoff |

**Operate Phase Staffing:** KGS (offshore) resources execute factory patterns at higher volume; onshore resources focus on quality assurance, escalations, and security classification.

### 6.3 Pre-Freeze Checkpoint (aligned with November production freeze)

By week 10 (approximately early November):
- All Tier 1 source families contracted and registered
- B2 (Bronze landing) has contracts needed to activate priority data pathways
- C (Silver pipelines) has contracts needed to begin Tier 1 transformations
- Executive checkpoint confirms readiness for pre-freeze data landing

---

## 7.0 Governance and Handoff

### 7.1 Unity Catalog Registration

Every completed contract is registered as governed metadata in Databricks Unity Catalog:
- Contract fields stored as table/schema properties and tags
- Ownership mapped to Unity Catalog access policies
- Classification drives row-level security and column masking enforcement
- Quality expectations configured as Delta Live Tables expectations
- Lineage visible in Unity Catalog lineage graph

### 7.2 Violation Workflows

Contracts are not static documents — they are enforced at runtime:
- **Schema drift detection:** Automated alert when incoming data deviates from contracted schema
- **SLA monitoring:** Freshness and completeness tracked per contracted threshold
- **Quality failure:** Data quarantined when quality expectations fail; ServiceNow incident opened
- **Ownership changes:** Triggered review when organizational changes affect data owners

### 7.3 BAU Process (post-engagement)

Upon workstream completion, KPMG transitions contract governance to CVS:
- **Contract maintenance playbook:** How to update contracts when sources change
- **New source onboarding template:** Enables CVS to author new contracts independently using established tooling
- **Quarterly review cadence:** Recommended governance rhythm for contract accuracy
- **Tooling handoff:** AI-assisted classification and schema inference tools remain available to CVS team

### 7.4 Relationship to Downstream Workstreams

| Consuming Workstream | What They Need from B1 | Gate |
|---|---|---|
| B2 (Bronze Landing) | Approved contract per source family → defines routing rules, table assignment, partition keys | Contract must be registered before source activates |
| C (Silver Pipelines) | Schema definition + lineage map → defines transformation logic and Silver table design | Contract must include downstream specification |
| D (Analytics Migration) | Dependency map showing which searches/dashboards depend on which source families | Disposition decision informs D prioritization |
| E (OCM) | Owner identification + classification → informs who needs training and on what data | Owner field and consumer list feeds persona mapping |

---

## 8.0 Alignment with SOW Scope and Deliverables

This methodology produces the following SOW-defined deliverables:

| SOW Deliverable | Methodology Section |
|---|---|
| Source Inventory and Rationalization Catalog | Step 1 (Discovery & Grouping) + Step 2 (Classification) |
| Source Family Taxonomy and Prioritization Framework | Steps 1–2 + Tier assignment |
| Data Contract Template and Contracting Standards | Section 3.0 (Contract Structure) |
| Completed Data Contracts for In-Scope Source Families | Steps 3–4 + Factory Model execution |
| Data Classification and Security Boundary Matrix | Step 2 (Classification scoring) + Contract Section: Security |
| Unity Catalog Metadata Registration Standards | Section 7.1 (Registration) |
| BAU Data Governance Handoff Materials | Section 7.3 (BAU Process) |

---

## 9.0 Staffing Coverage Summary

| Role | Focus Area | Engagement Model |
|---|---|---|
| Embedded Business Analyst(s) (1–3) | Domain/BU mapping, contract formalization, owner engagement support, process alignment to ODCS standards | Embedded with CVS contracts team; full duration |
| Data Architect (Director) | Contract standards, schema strategy, cross-workstream governance, Databricks ingestion bridging | Part-time; advisory and escalation |
| Security SME (Director) | PHI/PII/PCI classification, security boundary decisions, masking requirements, Cribl Guard alignment | 10 weeks focused on security-domain families |
| Partner/SciCom Engineer | Source family taxonomy alignment, Cribl routing coordination, Bronze architecture input | Ongoing collaboration |

This staffing model provides effective coverage because:
- CVS owns the contract completion process; KPMG's embedded analysts accelerate and formalize rather than build from scratch
- AI acceleration reduces per-contract effort by ~70%, maximizing embedded analyst throughput
- Domain-grouped workshops batch source-owner engagement to maximize throughput per session
- Data contracts are not on the critical path for migration — this allows the embedded team to operate at CVS's pace without blocking Workstream D (analytics migration)
- The bottleneck is source-owner response time (CVS dependency), not KPMG engineering capacity
