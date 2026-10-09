# CVS Environment Walkthrough for Workstream A Architect

**Purpose:** Onboarding reference for the KPMG Architect supporting Workstream A (Architecture & Use Case Validation)  
**Last Updated:** 2026-07-16  
**Classification:** KPMG Internal — Do Not Distribute to Client

---

## Table of Contents

1. [Engagement Context](#1-engagement-context)
2. [What Is Workstream A?](#2-what-is-workstream-a)
3. [CVS's Current State — The Big Picture](#3-cvss-current-state--the-big-picture)
4. [The 6-Project Program Structure](#4-the-6-project-program-structure)
5. [Ingestion Architecture (What's Already Built)](#5-ingestion-architecture-whats-already-built)
6. [Physical Network & Cloud Architecture](#6-physical-network--cloud-architecture)
7. [Databricks Environment (Current State)](#7-databricks-environment-current-state)
8. [Security & Governance Landscape](#8-security--governance-landscape)
9. [Data Scale & Source Complexity](#9-data-scale--source-complexity)
10. [The Source Rationalization Strategy](#10-the-source-rationalization-strategy)
11. [Target-State Architecture (What We're Proposing)](#11-target-state-architecture-what-were-proposing)
12. [Key People You'll Interact With](#12-key-people-youll-interact-with)
13. [Known Risks & Gaps You'll Assess](#13-known-risks--gaps-youll-assess)
14. [What "Good" Looks Like for WS-A](#14-what-good-looks-like-for-ws-a)

---

## 1. Engagement Context

CVS Health / Aetna is building a **large-scale telemetry data lake on Databricks** (Azure-based) to consolidate both **security (~50%) and observability (~50%)** data. The strategic objective is to **decouple data storage from analysis tools** — store everything cheaply in a lakehouse and retain freedom to send data to whatever runtime tool (CrowdStrike, Databricks SQL, Genie, future SIEM) makes sense per use case.

**Why they're doing this:**
- **Cost:** Splunk/Chronicle charge per-GB ingested. At confirmed 145.4 TB/day (Splunk Core production, top 126 source families) plus ~104 TB/day (Chronicle Phase 1), that's astronomical. Databricks + ADLS Gen2 is $/compute + cheap object storage — likely 5–10x cost reduction.
- **Vendor independence:** Data lives in *their* lake (ADLS Gen2), not locked in a vendor's proprietary format.
- **AI readiness:** Can't easily run ML/AI inside Splunk. Databricks is a native ML/AI platform.
- **Scale without linear cost growth:** Ingesting more data ≠ proportionally more cost in a lakehouse model.
- **Compliance & retention:** Immutable Bronze layer for provenance, lineage, chain of custody. WORM storage for regulatory requirements.

**Engagement structure:** KPMG + SciCom are co-delivering. Estimated total proposal is ~$8.4M. SciCom owns Cribl pipeline execution; KPMG owns architecture, governance, Silver/Gold design, analytics migration, security, and OCM.

---

## 2. What Is Workstream A?

### Objective
Validate the scalability and fitness of the existing Databricks architecture and execute one end-to-end use case as a reference implementation.

### Scope (12 weeks, ~$681K, 3,120 hours, ~6.5 FTE avg)

**Architecture Validation:**
- Assess the Azure / Cribl / Databricks architecture end-to-end
- Review Unity Catalog taxonomy and lifecycle configuration
- Validate the security model (RBAC, ABAC, row/column filtering)
- Validate data protection + DR posture
- Validate scalability for high-volume security and observability data (confirmed: 145.4 TB/day Splunk Core production; ES SH: 234 enabled correlations, 16 accelerated data models; Chronicle: ~104 TB/day Phase 1)
- Identify gaps and define target-state recommendations
- Conduct FinOps assessment (cluster policies, cost attribution)
- Validate Git + CI/CD approach

**Use Case Validation (Pilot):**
- Scope and prepare one high-value use case (likely security or K8s observability)
- Build Bronze & Silver pipelines for the pilot domain
- Model analytics (Gold layer, dashboards, AI/BI)
- Validate & showcase via demo

### Deliverables
- Architecture assessment report with gap analysis
- Target-state architecture and remediation plan
- FinOps and DataOps foundation recommendations
- Functional end-to-end demo (living pilot)
- Validated baseline that downstream workstreams (B, C, D) can build upon

### Why This Matters
The Databricks environment exists today but is described as a **"science project"** — strong governance/SDLC/Unity Catalog foundations were started, but there are **no usable outputs** (no working schemas, production tables, validated queries, or reports). A key architect recently departed with limited knowledge transfer. Your job is to assess what's actually there, determine what's production-ready vs. what needs remediation, and prove the architecture works end-to-end with one real use case.

---

## 3. CVS's Current State — The Big Picture

### The One-Line Summary
CVS has a well-engineered ingestion layer (Cribl + Azure Blob + Event Grid + Autoloader) being actively built, but **everything from Bronze forward is immature or nonexistent**. Your assessment covers the full stack.

### What Exists Today

| Component | Status | Maturity |
|---|---|---|
| Cribl infrastructure (AKS) | Being built (Projects 1–3) | High — in-flight, well-scoped |
| Azure Blob landing zones | Built | Operational |
| Event Grid / Event Hub | Configured | Functional |
| Databricks Autoloader jobs | Partially built | Needs production validation |
| Unity Catalog | Configured (basic) | Foundations only — needs full governance buildout |
| Bronze layer | Partially exists | Not production-validated; schema questions remain |
| Silver layer | **Does NOT exist** | This is the core gap |
| Gold layer | **Does NOT exist** | Depends on Silver |
| Detections / alerting | **Does NOT exist** | Future workstream (D) |
| User dashboards / queries | **Does NOT exist** | Future workstream (D/E) |

### What's In Flight (Not Your Problem to Build, But You Must Understand)
- **Cribl Worker deployment** (AKS-based, Azure) — go-live target September 19, 2026
- **Splunk Heavy Forwarder → Cribl Worker conversion** (38,000 UFs, ~150 HFs)
- **Network foundation** (Hub VNet, ExpressRoute, NRS Spoke VNet)
- **Public ingress edge** (Akamai + AppGw + F5 replacement)

---

## 4. The 6-Project Program Structure

CVS has organized the full initiative into 6 projects. The first 3 are scoped and in-flight. Projects 4–6 are where KPMG delivers:

| # | Project | Status | KPMG Role |
|---|---|---|---|
| 1 | Ingestion framework | In flight | **WS-A validates** |
| 2 | Cribl infrastructure in Azure | In flight | **WS-A validates** |
| 3 | Splunk HF → Cribl worker conversion | In flight | **WS-A validates** |
| 4 | Data movement / filling the lake / routing into Databricks | **Not scoped** | WS-B + WS-C build this |
| 5 | User migration, training, enablement | **Not scoped** | WS-E builds this |
| 6 | Security / SEM direction and integration | **Not scoped** | WS-D builds this |

**Your role (WS-A) spans validation of 1–3 and setting the architectural foundation for 4–6.**

### Planning Anchor
Infrastructure patterns from Projects 1–3 are expected to be available ~**September 15–19, 2026**. This is when data starts flowing through the new pipe and is the trigger for Project 4.

---

## 5. Ingestion Architecture (What's Already Built)

### End-to-End Ingestion Flow

```
Source Systems (Endpoint, Identity, Cloud, App, SaaS)
  → Azure Blob Storage (landing zone)
  → Cribl Workers (routing, forking, filtering, compression)
  → ADLS Gen2
  → Databricks Autoloader (triggered via Event Grid notifications)
  → Bronze (immutable raw, long-term retention)
  → Silver (curated, structured, analytics-ready)  ← DOES NOT EXIST YET
  → Gold (user-created analytics content)           ← DOES NOT EXIST YET
```

### 6 Defined Workflows

| WF | Path | Description |
|---|---|---|
| WF1 | DC → Azure Blob → ADLS Gen2 via Autoloader | Primary: 6 data centers ship via Cribl/NiFi over HTTPS 443 |
| WF2 | DC → Apache NiFi (Cribl Worker) → Blob → Databricks | Alternative DC path |
| WF3 | DC → Splunk HF → Cribl Worker → Public Internet → Blob | Legacy HF conversion path |
| WF4 | GCP (Pub-Sub/GCS) → Cribl Worker (GKE) → Azure Blob | Multi-cloud: Google sources |
| WF5 | AWS (CrowdStrike → S3/Kinesis → SQS → Cribl ECS) → Azure Blob | Multi-cloud: AWS sources |
| WF6 | Azure (Event Hub/Blob) → Cribl Worker (AKS) → Databricks | Azure-native log sources |

### Key Architecture Decisions Already Made
- **Cribl** deal closed; build underway
- **Single Azure tenant** (confirmed)
- **Event Grid** = file notification/alerting (avoids full directory scans in Autoloader)
- **Event Hub** = queue/ingestion for Azure-native logs and future consumers
- **Zero Bus** = future option (Cribl → Databricks Bronze directly, removes storage tier) — not day-one
- **Latency SLA:** 99.9% ingested within 5 minutes of generation
- **PHI/PII masking at Cribl** (before data reaches the lake) with 40–60% volume reduction via filtering

---

## 6. Physical Network & Cloud Architecture

### WF1 Physical Topology (Primary Pattern — Understand This Well)

```
ON-PREMISE (6 Data Centers)
  Cribl Worker / NiFi
  - NVMe Persistent Queue (2 TB per site)
  - Certificate-based Service Principal auth
  - No public IP on workers
        │
        ▼
DC PERIMETER
  - Firewall + NAT
  - Outbound TCP 443 only (TLS 1.2+)
  - Static NAT IP per site (critical dependency — if dynamic, ingestion silently fails)
        │
        ▼
AZURE BLOB (PRIMARY — EAST US 2)
  - Storage V2 Premium (ZRS)
  - Container: dc-logs-raw
  - Public endpoint (IP ALLOWLIST ONLY — 6 DC static NAT IPs)
  - CMK-HSM encryption at rest
        │
        ▼
EVENTING
  - Event Grid (system topic, BlobCreated events)
  - Event Hub (Standard tier, dedicated consumer group)
  - DR fallback: directory listing mode (5 min polling)
        │
        ▼
DATABRICKS
  - Auto Loader cluster (streaming)
  - VNet injection (no public IP)
  - Access via Service Endpoint + Private Endpoint
  - mergeSchema = true
  - <10s ingestion lag per diagram notes
        │
        ▼
DATA LAKE (ADLS Gen2)
  - Bronze: RESTRICTED (PHI present)
  - Silver: INTERNAL (PHI masked)
  - Private endpoint only (10.100.9.4)
  - No public access
        │
        ▼
CONSUMERS
  - Databricks analytics (Unity Catalog governed)
  - Optional SIEM fan-out (Sentinel / Splunk / Chronicle)
```

### Network Security Model

| Boundary | Control |
|---|---|
| Blob Ingress | IP allowlist (6 static DC NAT IPs + Databricks subnet via service endpoint). Everything else denied. |
| DC Egress | TCP 443 → AzureCloud tag only. All other outbound Azure traffic denied. |
| Data Lake Access | Private endpoint ONLY (10.100.9.4). No public access. |
| Internal Transit | Istio mTLS between all services; TLS 1.3 minimum |
| External Edge | Akamai (DDoS, WAF, bot protection) for SaaS/external sources |
| Workload Isolation | Secure network 10.100.0.0/20 using Azure Firewall + private-only endpoints |

### Identity Model

| Identity | Scope | Auth | Rotation |
|---|---|---|---|
| Writer SP (DC → Blob) | Blob Data Contributor (container-scoped) | Certificate-based (CVS PKI) | 90-day |
| Reader/Processor (Auto Loader) | Blob Data Reader + ADLS Bronze Contributor + Event Hub Receiver + Unity Catalog | Managed Identity (preferred) | Automatic |
| Writer SP ≠ Reader SP | Least privilege enforced | — | — |

### Data Classification Boundary (Critical for Your Assessment)

```
Bronze → RESTRICTED (PHI present, raw, unmasked)
Silver → INTERNAL (PHI masked via DLT pipeline)
```

This classification boundary is **enforced before analytics access**. If masking fails, PHI leaks into Silver. This is a key risk to validate in your assessment.

### Backpressure Architecture (3-Zone Model)

The system uses multi-layer buffering to decouple failures:

| Zone | Component | Mechanism | Buffer Window |
|---|---|---|---|
| Zone 1 (Source) | Cribl Worker PQ | NVMe 2TB, block-on-full policy, exponential retry | 58 min (Shea/RI) to 4.4 hrs (Vegas) |
| Zone 2 (Blob) | Azure Blob throttling | Short burst absorbed by PQ; sustained → split storage accounts | SEV2 at 15 min, SEV1 at 30 min |
| Zone 3 (Processing) | Databricks / ADLS | Auto Loader scale-out (4→12 workers); partition by dc_site + event_date | Elastic |

**Critical Risk:** Shea (RI) data center has < 1 hour PQ buffer. If outage persists beyond that, upstream throttling triggers.

### DR Architecture

- **Blob GRS replication** to US Central
- **Auto Loader DR fallback:** directory listing mode (5 min polling vs. Event Grid real-time)
- **Rollback model:** Parallel-run architecture. Splunk pipeline stays active. Rollback = configuration toggle in Cribl, NOT data recovery.
- **Per-site rollback supported.** Recommended go-live order: Atlanta → Vegas → Windsor → Middletown → Arizona → Shea (lowest buffer capacity last).

---

## 7. Databricks Environment (Current State)

### Workspace Inventory

| Cloud | Count |
|---|---|
| AWS | 10 |
| Azure | 125 |
| Azure — Not Running | 66 |
| **Total** | **201** |

This is the enterprise Databricks footprint you'll be assessing. The observability/security data lake is a subset of these workspaces.

### What SciCom (Raj/Amar) Has Built So Far
- Databricks workspace(s) provisioned in Azure
- Unity Catalog configured (basic level)
- Some Autoloader jobs exist (not production-validated)
- CI/CD pipeline exists (needs validation — Git integration approach)
- Governance foundations started but not operationalized

### What's Missing (Per Client Feedback)
- No production-ready schemas
- No working Silver tables
- No validated queries or reports
- No detection pipelines
- No user-facing analytics
- Key architect departed with limited KT
- Platform described as "science project" — strong vision, no usable outputs

### Cribl Platform Timeline (Affects When Data Flows to You)

| Phase | Name | Dates |
|---|---|---|
| P5 | AKS Cluster + Cribl Leader | Jun 23 – Jul 4 |
| P6 | Cribl Worker Clusters (DC + Cloud) | Jul 7 – Jul 25 |
| P7 | WF1 Tanium + WF6 Azure EH Canary | Jul 28 – Aug 8 |
| P8 | WF2–5 Full Rollout + Tanium Ramp | Aug 11 – Sep 5 |
| P9 | Ops Hardening — Runbooks, Monitoring, DR | Sep 8 – Sep 19 |

**Total: Go-Live September 19, 2026** — this is when real data starts flowing through the new pipelines into Databricks.

---

## 8. Security & Governance Landscape

### Security Assessment Already Underway (Lovelytics)
A separate security assessment by Lovelytics is in progress. Their scope (from Josh Sparaga's definition) includes:
- Enhanced Security Monitoring as required for all workspaces
- Automatic Cluster Updates policy enforcement
- Company security tooling integrated into Databricks base images
- Automation to detect non-compliant workspaces
- Discovery & workspace inventory validation (all 201 workspaces)
- Baseline security hardening (Tranche 1)
- Upstream image and runtime supply chain security
- V1 control set definition

**Your overlap concern:** The Lovelytics assessment may produce findings that overlap with your WS-A scope. Coordinate early to avoid duplication.

### Target Security Controls (V1 Hardening)

| Domain | Controls |
|---|---|
| Identity | Enterprise identity integration; lifecycle management; no local auth |
| Unity Catalog | Governance, lineage, centralized access control |
| Network | Private Link only; no public IPs; VNet injection |
| Compute | Approved runtimes (LTS); cluster policy enforcement |
| Monitoring | Audit logging; SIEM integration; enhanced security monitoring |
| Secrets | Key Vault integration; managed rotation |
| CI/CD | Infrastructure-as-Code; policy-as-code; CI/CD gates |
| Cost | Tagging; cluster policies; usage accountability |

### Compliance Requirements
- **HIPAA** — PHI data classified as RESTRICTED in Bronze; must be masked before Silver
- **PCI-DSS** — payment card data handling
- **WORM storage** for long-term compliance retention (<10% of data)
- **100% TLS 1.2+** (1.3 preferred); CMK at rest; Key Vault managed keys
- **Centralized RBAC, lineage, audits** via Unity Catalog

---

## 9. Data Scale & Source Complexity

### Volume

| Metric | Value |
|---|---|
| Total daily volume | 300–400 TB/day |
| Split | ~50% security, ~50% observability |
| Sustained throughput | 4.63 Gb/s |
| Peak throughput | 16.68 Gb/s |
| Events per day | ~207 billion |
| Universal Forwarders | 38,000 |
| Heavy Forwarders | ~150 |
| Splunk Indexes | 925 |
| Source types | 12,910 (raw Splunk source types) |
| Daily Splunk users | ~10,000 |
| Security data sources | 170+ |
| CI types | 6,000+ |
| Internet-facing devices | 200–400K |

### Source Complexity (The "12K Problem")
The original framing of "20+ data sources" is materially understated. Reality:
- **12,910 raw Splunk source types** — but most are Kubernetes deployment artifacts, not unique schemas
- **~10,790 are Kubernetes** (84% of all source types) — dynamically generated deployment identifiers
- A single "source" (e.g., CRM) may have hundreds of source types across environments/pods/regions
- Chronicle alone has ~160 unique sources
- Total may be thousands of unique data streams

### User Populations (Who Will Consume the Lake)

| Persona | Count | Behavior |
|---|---|---|
| Security Analysts | ~100 | SOC, threat hunters, IR. Highest-touch. |
| Power Users | ~150 | Dashboard builders, advanced SPL authors |
| Observability Consumers | ~7,000 | Look at dashboards, run pre-built searches |
| Occasional / Read-Only | ~2,750 | Monthly reporters, managers, audit |

Key insight: Majority of queries target **recent data (0–30 days)**. A small subset (~100 core metrics) drives most usage.

---

## 10. The Source Rationalization Strategy

This is critical context for your pilot design. The approach (developed by Sid Roy at SciCom) compresses the unmanageable source type sprawl:

### The Funnel

```
12,910 raw source types
    → Pattern recognition (strip deployment metadata)
    → Application family extraction
    → ~250-300 logical source families (50:1 ratio)
    → ~12 domain categories
    → ~25 governed Bronze tables
    → 20-30 Silver business domain tables
```

### Kubernetes Example (84% of Sources)

```
kube:container:crm-api-prod     ┐
kube:container:crm-api-dev      │
kube:container:crm-api-qa       ├─→ Logical Family: "CRM Applications"
kube:container:crm-api-east     │       → bronze_crm (single table)
kube:container:crm-api-west     ┘

Deployment metadata becomes COLUMNS, not separate tables:
  deployment_name, namespace, cluster, environment, region, pod, container, app_version, image, raw_event
```

Result: **10,790 K8s source types → ~40 logical Kubernetes platforms**

### 12 Domain Categories

| Domain | Est. Families | Examples |
|---|---|---|
| K8 Application Platform | ~20 | CRM, RXDW, RXBE, Aetna, Eligibility, Pharmacy AI |
| Network & Security | ~40–60 | Cisco, Palo Alto, F5, Citrix, Fortinet, Check Point |
| VMware Infrastructure | ~10–20 | ESXi, vCenter, vMotion, DRS |
| Windows Platform | ~15–20 | WinEventLog, AD, DNS, PowerShell, Sysmon |
| Linux / Unix | ~15 | Syslog, Auditd, SSH, Systemd |
| Web & Middleware | ~20 | Apache, IIS, Tomcat, DataPower, NGINX |
| Cloud Providers | ~20 | Azure, AWS, GCP (activity logs, audit, flow logs) |
| SaaS Platforms | ~20 | CrowdStrike, Okta, Workday, ServiceNow, Zscaler |
| Databases | ~20 | Oracle, SQL Server, DB2, MongoDB, Redis |
| Enterprise Applications | ~60–80 | Pharmacy, Claims, Loyalty, PBM, Supply Chain |
| Messaging & Streaming | ~10 | Kafka, IBM MQ, Azure Event Hub |
| Observability & Monitoring | ~20 | Splunk Forwarders, Cribl, Prometheus, Datadog |

### Governance via ODCS (Open Data Contract Standard)
Every logical source family gets an ODCS contract defining:
- Data owner (business + technical)
- Canonical schema
- Security classification
- Quality rules and SLAs
- Retention policy
- Downstream consumers
- Lineage declarations

---

## 11. Target-State Architecture (What We're Proposing)

### Medallion Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                 UNITY CATALOG (Governance Layer)                 │
│  RBAC • Lineage • Audit • Classification • Dynamic Masking      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BRONZE (~25 governed tables)                                   │
│  - Immutable raw ingest from logical source families            │
│  - Time-partitioned (ingestion_time, source_type, source_file)  │
│  - Zero transformation — preserves auditability and replay      │
│  - WORM/cold tier for long-term retention                       │
│  - Data classification: RESTRICTED (PHI present)                │
│                                                                 │
│  SILVER (20–30 business domain tables)                          │
│  - Parsed, flattened, schema-enforced                           │
│  - OCSF alignment for security data                             │
│  - Enrichment joins (CMDB, identity, GeoIP, threat intel)       │
│  - Data classification: INTERNAL (PHI masked)                   │
│  - Governed by ODCS contracts                                   │
│                                                                 │
│  GOLD (curated business products)                               │
│  - Aggregations, KPIs, detection outputs                        │
│  - User-created analytics (per workspace, promotable to Silver) │
│  - Dashboards, alerts, compliance reports                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Proposed Workspace / Catalog Strategy

```
Unity Catalog (Metastore)
├── catalog: observability_prod
│   ├── schema: bronze
│   ├── schema: silver
│   └── schema: gold
├── catalog: security_prod
│   ├── schema: bronze
│   ├── schema: silver
│   └── schema: gold
├── catalog: shared_enrichment
│   ├── schema: identity
│   ├── schema: assets (CMDB)
│   ├── schema: threat_intel
│   └── schema: geoip
└── catalog: workspaces_user
    ├── schema: sre_team
    ├── schema: security_ops
    └── schema: app_team_*
```

### Compute Architecture

| Resource | Purpose | Model |
|---|---|---|
| Autoloader Cluster | Continuous ingestion ADLS → Bronze | Always-on Structured Streaming; auto-scales |
| DLT Pipeline Cluster | Bronze → Silver transformation | Lakeflow Declarative Pipelines; streaming + scheduled |
| SQL Warehouse (Serverless) | Ad-hoc queries by 7–10K users | Serverless auto-scaling; pay-per-query |
| Genie / AI Serving | NL querying, anomaly detection | Model Serving endpoints; Mosaic AI |
| Detection Jobs | Scheduled + streaming detections | Lakeflow Jobs; parameterized SQL/PySpark |
| Notebook Clusters | Power user investigation, hunting | On-demand interactive; per-workspace isolation |

### Consumer Integration

| Consumer | Method | Data |
|---|---|---|
| Observability Users (7–10K) | SQL Warehouse + Genie | Silver + Gold via dashboards |
| CrowdStrike (likely SIEM) | Event Hub / REST push from Gold | Curated alerts, enriched events |
| BI / Reporting (Power BI) | Databricks SQL connector | Gold KPIs |
| Data Science / ML | Mosaic AI, Model Serving | Silver for training |
| Ticketing (ServiceNow) | REST API from detection pipeline | Alert → ticket |

---

## 12. Key People You'll Interact With

### CVS / Aetna (Client)

| Name | Role | Relevance to WS-A |
|---|---|---|
| Joshua Sparaga | Program/architecture lead | Your primary day-to-day client contact. Sets scope expectations. |
| Chris Robinson | Lead Director, Obs Engineering | Runs engineering; owns the 6-project structure. Gave critical feedback on architecture maturity. |
| Shivpratap Singh | Ingestion pipeline SME | Knows Event Grid, Event Hub, Autoloader details. Key for validation questions. |
| Matthew Filippone | Technical SME | Autoloader optimization, file ingestion specifics. |
| Ken O'Connor | Executive sponsor | Final decision-maker. You likely won't interact directly. |
| Ryan Budnik | Security stakeholder | Security/SEM direction. Relevant for security assessment overlap. |

### SciCom (Partner — They Built What You're Assessing)

| Name | Role | Relevance to WS-A |
|---|---|---|
| Sid Roy | VP Operations | Primary SciCom contact. Developed the source rationalization taxonomy. Strategic thinker. |
| Raj (Amar) | Data/Platform Engineer | **Owns the Databricks environment.** Governance, pipelines, ingestion jobs. Will provision your access. |
| Peter Cherrick | Sr Enterprise Outsourcing Mgr | Day-to-day SciCom delivery coordination. |

### KPMG Team

| Name | Role | Relevance to WS-A |
|---|---|---|
| Deepak Benjamin | WS-A Owner / Architecture | Your lead. Has Toyota/Ford reference architectures. |
| Niels Hanson | Lead Databricks Engineer | Technical lead across the proposal. AI-assisted metadata concept. |
| Danish Markatia | Data Architecture / WS-C owner | Silver layer design. His workstream depends on your outputs. |
| Thomas Haslam | Relationship lead | Proposal owner; coordinating with client and SciCom. |
| Ryan Budnik | Security (KPMG side) | Security workstream lead. |

---

## 13. Known Risks & Gaps You'll Assess

### Architecture Risks (Confirmed)

| Risk | Detail | Severity |
|---|---|---|
| Key architect departed | Limited knowledge transfer to remaining team | HIGH |
| Platform functionally immature | No production readiness validation done | HIGH |
| Security architecture underdeveloped | Per Chris Robinson's feedback (Jun 25) | HIGH |
| Lovelytics overlap | Their security assessment may conflict with your scope | MEDIUM |
| PHI masking failure risk | If DLT masking fails, PHI leaks from Bronze → Silver | HIGH |
| Schema evolution | mergeSchema=true may cause pipeline failures at scale | MEDIUM |
| Shea DC buffer < 1 hour | Sustained outage causes throttling at highest-volume site | MEDIUM |
| Dynamic NAT IP risk | If DCs switch to dynamic NAT, ingestion silently fails (403) | HIGH |
| No production readiness for Autoloader jobs | Exist but not validated under load | HIGH |

### Open Design Decisions (Yours to Resolve/Recommend)

| ID | Decision |
|---|---|
| OI-001 | Confirm static NAT IPs for all 6 DCs |
| OI-002 | NiFi vs Cribl for WF1 (impacts PHI handling) |
| OI-003 | Latency SLA: <30s vs. 5 min (affects compute cost) |
| OI-004 | DC egress capacity (can it sustain full throughput?) |
| OI-005 | Auto Loader sizing for ~80 TB/day per cluster |
| OI-006 | PHI masking validation approach (how to prove it works) |
| OI-007 | Unity Catalog ACL enforcement model (who gets what) |
| OI-008 | Shea PQ capacity upgrade (is 2TB enough?) |

### Specific Things to Validate

1. **Does the Autoloader actually work at scale?** — Test at realistic volume. Check checkpoint behavior, schema inference, exactly-once guarantees.
2. **Is Unity Catalog properly configured?** — Check metastore binding, catalog permissions, lineage tracking, audit logging.
3. **Can the PHI masking pipeline guarantee no leakage?** — Test edge cases, schema evolution scenarios, late-arriving fields.
4. **Is the Git/CI/CD pipeline production-grade?** — Check branching strategy, deployment gates, rollback capability.
5. **Are cluster policies in place?** — Cost controls, runtime restrictions, autoscaling limits.
6. **Is DR tested?** — GRS replication, Auto Loader fallback mode, rollback procedures.
7. **Does FinOps exist?** — Cost attribution, tagging, usage monitoring, budget alerts.

---

## 14. What "Good" Looks Like for WS-A

### Week 1–3: Discovery & Access
- Get provisioned into the Databricks environment (Raj owns this)
- Inventory what exists: workspaces, catalogs, schemas, tables, jobs, pipelines, CI/CD repos
- Document current state vs. expected state
- Identify what the departed architect left behind

### Week 4–6: Assessment & Gap Analysis
- Run through the security/governance checklist against all 201 workspaces (coordinate with Lovelytics)
- Validate Autoloader under realistic load
- Stress-test PHI masking pipeline
- Assess CI/CD maturity and Git integration
- FinOps review: compute costs, cluster policies, tagging

### Week 7–10: Pilot Build
- Select one high-value domain (likely K8s/CRM or Network Security)
- Build Bronze → Silver → Gold for that domain
- Implement ODCS contract for the pilot source family
- Create a working dashboard or Genie demo

### Week 11–12: Recommendations & Handoff
- Architecture assessment report with prioritized remediation backlog
- Target-state architecture diagram (validated, not theoretical)
- Remediation plan with sequencing and effort estimates
- Handoff brief to WS-B, WS-C, WS-D teams

### Success Criteria
- Downstream workstreams (B, C, D) can start building on your validated foundation without rework
- Client has confidence the architecture can handle 400 TB/day
- Security team has confidence PHI controls are effective
- FinOps baseline is established (cost per TB ingested, cost per query, etc.)
- One end-to-end use case is running in production (even if limited scope)

---

## Appendix: Key Technical Specs at a Glance

| Spec | Value |
|---|---|
| Cloud | Azure (primary), limited AWS (10 workspaces) |
| Region | East US 2 (primary), US Central (DR) |
| Storage | ADLS Gen2 + Azure Blob |
| Compute | Databricks (VNet-injected, no public IP) |
| Ingestion | Cribl Workers (AKS) + Databricks Autoloader |
| Governance | Unity Catalog |
| Encryption | TLS 1.2+ in transit; CMK-HSM at rest; Key Vault |
| Network | 10.100.0.0/20; Azure Firewall; Private Endpoints only |
| Auth | Managed Identity (preferred) + Certificate-based SPs |
| SIEM (likely) | CrowdStrike (decision pending) |
| Compliance | HIPAA, PCI-DSS, FedRAMP |
| Availability | 99.9% monthly; zero-downtime upgrades |
| Burst | +50% headroom; scale-out within 10 minutes |
| Data Quality | 98% pass rate target |
| Monthly Cost (WF1 infra) | ~$5.7K–$11K |

---

## Appendix B: Document References

The following documents contain detailed source material referenced throughout this walkthrough. All paths are relative to the repository root.

### Architecture & Infrastructure

| Document | Path | What It Covers |
|---|---|---|
| WF1 Physical Architecture (DC → Blob → ADLS) | `resources/text-extracts/SAIPA-WF1 — DC to Azure Blob to ADLS Gen2 via Databricks Auto Loader-010626-1235.txt` | Full 7-layer physical architecture, network topology, identity model, backpressure zones, rollback architecture, risk model, and open design decisions |
| WF2 Multi-CSP Architecture | `resources/text-extracts/SAIPA-WF2 — Multi-CSP Native Logs (GCP · Azure · AWS) to Azure Blob to ADLS Gen2.txt` | GCP, Azure, and AWS native log ingestion paths |
| WF3 External SaaS Architecture | `resources/text-extracts/SAIPA-WF3 — External SaaS (Tanium) → Akamai Edge (mTLS + WAF) → Azure Blob → Cri.txt` | Tanium / external SaaS ingestion via Akamai edge |
| Logical Architecture (Target State) | `resources/text-extracts/SAIPA-Observability Data Lake Logical Architecture -Target State-010626-123513.txt` | Target state logical architecture diagram (image-only — refer to PDF) |
| Databricks Security Lake Architecture | `resources/databricks-security-data-lake-example-architecture.m` | Reference architecture diagram (PNG image) |
| Cloud & Databricks Architecture (Internal) | `01_internal/Thoughts-on-Cloud-and-Databricks-Architecture_2026-6-15.md` | Detailed proposed architecture: compute model, workspace strategy, data flow patterns, security controls, consumer integration, phasing, and Databricks platform features |

### Cribl & Ingestion Program

| Document | Path | What It Covers |
|---|---|---|
| Cribl Program Plan (Summary) | `resources/text-extracts/Cribl_Program_Plan_v3_PBR-Copilot-summary.txt` | Cribl deployment timeline, phases P0–P9, go-live September 19 |
| Cribl Program Plan (Full) | `resources/text-extracts/Cribl_Program_Plan_v3_PBR.txt` | Detailed Cribl infrastructure plan |
| Observability Pipelines Leadership Deck | `resources/text-extracts/Observability_Pipelines_Leadership_Deck_4_Exec_Copilot-extract.txt` | Executive deck on pipeline architecture and delivery targets |

### Source Rationalization & Data Model

| Document | Path | What It Covers |
|---|---|---|
| Source Type Rationalization Overview | `resources/2026-7-13_proposal-from-sid/CVSDatabricksSourceTypeRationalizationOverview.txt` | Full rationalization framework: 12,910 → 250-300 families, domain categories, ODCS contracts, target lakehouse architecture |
| K8s Rationalization Deep-Dive | `resources/2026-7-13_proposal-from-sid/CVSSourceTypeRationalizationK8.txt` | Kubernetes-specific rationalization (10,790 → 40 families), pattern recognition methodology, Bronze/Silver modeling |
| Workstream Costing (Revised) | `resources/2026-7-13_proposal-from-sid/2026-7-13_workstream-costing.txt` | Revised staffing and costing for all workstreams (B1, B2, C, D) including source-family taxonomy impact |

### Security & Governance

| Document | Path | What It Covers |
|---|---|---|
| Security Scope (from Josh Sparaga) | `02_client-meetings/2026-6-09_Security-scope-from-Josh.txt` | Lovelytics security assessment scope: workspace inventory, hardening baseline, runtime supply chain, V1 control set, policy exceptions |
| Security Lakehouse Blueprint | `resources/text-extracts/2025-09-eb-building-a-security-lakehouse-blueprint-v1-ss-191609.txt` | Databricks security lakehouse reference architecture and best practices |

### Client Meetings & Context

| Document | Path | What It Covers |
|---|---|---|
| Initial Touchpoint with Client | `02_client-meetings/2026-6-10_Initial-touchpoint-with-client.md` | First architecture walkthrough: 6-project structure, ingestion flow, Event Grid/Hub, Bronze/Silver/Gold, data contracts, AI-assisted metadata, proposal framing |
| KPMG-SciCom-Databricks Approach (Jun 23) | `02_client-meetings/2026-6-23_KPMG-Scicom-Databricks-approach-cvs.txt` | Client-facing approach discussion |
| KPMG-SciCom-Databricks Approach (Jun 24) | `02_client-meetings/2026-6-24_KPMG-Scicom-Databricks-approach-cvs.txt` | Follow-up approach refinement |
| Client Connect on Proposal Feedback | `02_client-meetings/2026-7-8_Client-connect-on-proposal-feedback.txt` | Client feedback on proposal direction |
| Daily Connect incl. CVS (Jun 26) | `02_client-meetings/2026-6-26_daily-connect-including-cvs.txt` | Confirmed data scale (38K UFs, 150 HFs, 925 indexes) |

### Proposal & Staffing

| Document | Path | What It Covers |
|---|---|---|
| Proposal Deck Text (Jun 16) | `resources/proposal-deck-text_2026-6-16_528.txt` | Workstream definitions (A–F), objectives, activities, outcomes, timeline |
| Working Analysis (Jun 26) | `resources/working-analysis_2026-6-26.md` | Staffing model, budget math, build/operate factory model, capacity assumptions, user onboarding model |
| Master Context | `00_context/master-context.md` | Single source of truth: opportunity summary, stakeholders, architecture, program structure, data scale, timeline |
| Existing Architecture Timeline | `resources/2026-6-22_existing-arch-timeline` | Phase 0–4 timeline from mobilization through cutover and hypercare |
| FY26 Labor Costs | `resources/fy26-labor-costs.txt` | KPMG rate card (internal cost/hr by level) |

### Internal Strategy & Notes

| Document | Path | What It Covers |
|---|---|---|
| Niels Working Notes | `resources/niels-notes-questions_WORKING.md` | 400+ lines of technical questions, observations, and working hypotheses |
| Discussion with Sid on Proposal (Jul 13) | `01_internal/2026-7-13_discussion-with-cid-on-proposal.txt` | Key decisions on source-family taxonomy, pod structure, pricing reductions |
| Feedback from Chris via Sid | `01_internal/2026-6-25_Feedback-from-Chris-via-Sid.txt` | Critical architecture feedback — security underdeveloped, platform immature |
| Splunk-to-Databricks Migration Use Case | `01_internal/2026-6-12_Splunk-to-Databricks-Migration-Use-Case.txt` | Use case framing for the migration |

### External References

| Document | Path | What It Covers |
|---|---|---|
| Databricks Lakewatch Announcement | `resources/Databricks Announces Lakewatch_ New, Agentic SIEM _ Databricks Blog.html` | Databricks' agentic SIEM product (potential future consolidation path for CVS) |
| Chronicle Datasource Roadmap | `resources/text-extracts/Chronical_Roadmap_Datasources.txt` | Chronicle's 160 source types — context for multi-SIEM landscape |
| Data Lake Sync Notes (May 20) | `resources/text-extracts/Weekly_Program_Data_Lake_Sync_19May26.txt` | Weekly program sync — early program status |
