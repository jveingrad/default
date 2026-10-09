# Workstream B2: Databricks Bronze Data Landing — Detailed Methodology

## 1.0 Approach Overview

This document describes the detailed methodology KPMG will employ to validate and operationalize data pathways from Cribl and Azure Data Lake Storage (ADLS) Gen2 into Databricks Bronze Delta tables. It supplements SOW Section 2.4 (Workstream B2) by providing the specific validation approach, workflow-by-workflow readiness criteria, Bronze table design methodology, throughput testing plan, and pre-freeze go/no-go framework.

**The Problem:**
CVS ingests approximately 300–400 TB/day of telemetry across 6 data centers, 3 cloud providers, and dozens of SaaS platforms. This telemetry must flow through Cribl Stream, land in ADLS Gen2, and be continuously ingested into Databricks Bronze Delta tables — all within a 5-minute end-to-end SLA. The Bronze layer must consolidate approximately 925 Splunk indexes into ~25–50 governed Delta tables while preserving complete source fidelity.

**Confirmed Volume Baseline:**
SciCom's Technical Implementation Approach (v1.0, Jul 26, 2026) validated the top 2,000 production sourcetypes covering **145.4 TB/day** across **126 physical source families**. This represents ~98% of production volume and forms the Tier 1 ingestion scope. The remaining enterprise telemetry (long-tail sources, ES-specific feeds, and Chronicle security data at ~104 TB/day) extends the total toward the 300–400 TB/day enterprise estimate. The Tier 1 sources are the priority for pre-freeze Bronze landing validation.

**The Objective:**
Validate that every ingestion workflow (WF1–WF6) delivers data reliably into Bronze Delta tables at production scale. Confirm Bronze table design, Autoloader configuration, Event Grid notifications, and fork routing rules. Activate Tier 1 priority sources before the November production freeze.

**Delivery Boundary — KPMG / SciCom:**

| SciCom Owns (Executes) | KPMG Owns (Validates & Designs) |
|---|---|
| Cribl worker deployment (6 DCs + cloud) | Bronze table schema and partition design |
| Cribl routing pipeline configuration | Autoloader and Event Grid configuration |
| AKS/networking infrastructure | Throughput and SLA validation testing |
| Persistent queue management | Fork routing strategy and readiness criteria |
| Operational runbooks for Cribl | Pre-freeze readiness checkpoint and go/no-go |

**Relationship to Other Workstreams:**
- Workstream A validates the platform architecture that B2 operates within
- Workstream B1 provides data contracts that define which sources route to which Bronze tables
- Workstream C consumes Bronze tables as input for Silver/Gold pipelines — C cannot start transformation until B2 confirms data is landing
- Workstream D depends on Bronze data availability to begin analytics migration testing

**Architecture Baseline:**
This methodology references the Enterprise Telemetry Pipeline Architecture defined in the Technical Implementation Approach (Scicom, v1.0, Jul 26, 2026 — Sections 6–7) as the authoritative design baseline. KPMG's role is to validate, test, and operationalize that design rather than redesign it.

---

## 2.0 Ingestion Architecture

### 2.1 End-to-End Pipeline

The telemetry pipeline follows a layered architecture in which each component performs a distinct function:

```
Enterprise Sources (Apps, K8s, VMware, Windows, Linux, Cloud, Network, Security, DBs, SaaS)
    │
    ▼
Cribl Stream Workers (collect, parse, enrich, route, compress, guard)
    │
    ▼
Azure Data Lake Storage Gen2 — Enterprise Landing Zone
    │
    ▼
Databricks Auto Loader (streaming ingestion via Event Grid notifications)
    │
    ▼
Enterprise Bronze Delta Tables (~25–50 governed tables)
    │
    ▼
Silver Business Domains → Gold Business Products
```

**Key design principle:** Cribl optimizes telemetry transport and maintains source fidelity. It does NOT perform business transformations — that responsibility belongs to the Silver layer (WS-C). Bronze preserves the original event payload without modification.

### 2.2 Cribl Stream Functions

Cribl Stream performs the following pipeline functions before data reaches ADLS:

| Function | Purpose |
|---|---|
| Source Classification | Identify approved source families and assign Bronze destinations |
| Metadata Enrichment | Add Line of Business, Application, Environment, Region, Ownership |
| Timestamp Normalization | Standardize timestamps and time zones |
| Payload Validation | Validate event completeness and required attributes |
| Routing | Route telemetry to the appropriate ADLS Gen2 landing path |
| Compression | Reduce network utilization and storage consumption |
| Filtering | Remove duplicate or non-actionable telemetry where approved |
| Sensitive Data Protection | Mask or tokenize regulated fields where required (via Cribl Guard) |
| Persistent Queueing | Buffer telemetry during downstream outages; support replay |

### 2.3 ADLS Gen2 Landing Paths

Landing folders are aligned with Bronze Data Products rather than Splunk indexes or sourcetypes:

| ADLS Gen2 Path | Bronze Data Product | Primary LOB |
|---|---|---|
| `/bronze/cloud/platform/` | Cloud Platform Telemetry | Cloud Platform |
| `/bronze/security/network/` | Enterprise Network Security | Infrastructure & Security |
| `/bronze/security/windows/` | Windows & Endpoint Events | Infrastructure & Security |
| `/bronze/infrastructure/vmware/` | VMware Infrastructure | Infrastructure |
| `/bronze/pharmacy/rxconnect/` | RxConnect Platform | Rx Pharmacy |
| `/bronze/pharmacy/ivr/` | Rx IVR Platform | Rx Pharmacy |
| `/bronze/digital/kubernetes/` | Kubernetes Applications | Digital & Consumer |
| `/bronze/integration/` | Enterprise Integration Services | Aetna |
| `/bronze/clinical/enterprise_health/` | Enterprise Health | Clinical |
| `/bronze/audit/enterprise/` | Enterprise Audit | Security & Compliance |

### 2.4 Landing File Standards

| Attribute | Standard |
|---|---|
| Landing Format | JSON (raw) or Parquet where source supports structured output |
| Compression | Snappy or GZIP |
| Target File Size | 128–512 MB |
| Folder Hierarchy | Bronze Product / Year / Month / Day / Hour |
| Encryption | Azure Storage Encryption (CMK via Key Vault) |
| Lifecycle Policies | Automated retention and archival |
| Immutable Storage | Optional for regulated workloads (WORM) |

---

## 3.0 Workflow-by-Workflow Validation

### 3.1 Validation Framework

Each of the 6 ingestion workflows follows the same 5-step validation sequence:

```
Step 1: Connectivity Validation
    Confirm network path, ports, authentication, and endpoint availability

Step 2: Sample Data Validation
    Inject small-volume test data; confirm it lands in correct ADLS path
    with expected format, compression, and metadata

Step 3: Schema Validation
    Verify landed files match Bronze canonical schema expectations
    (required fields present, data types correct, raw_payload intact)

Step 4: Throughput Testing
    Ramp to production-representative volume; measure end-to-end latency
    against 5-minute SLA (see Section 7.0)

Step 5: SLA Confirmation
    Sustained run at target volume for 24+ hours; confirm 99.9% of events
    meet 5-minute ingestion SLA; document any failures
```

### 3.2 WF1 — Tanium (DC → Akamai → Blob → Autoloader)

| Attribute | Detail |
|---|---|
| **Source** | Tanium Stream (endpoint telemetry) |
| **Volume** | ~68 TB/day (largest single source) |
| **Path** | Tanium SaaS → Akamai Edge (mTLS + WAF) → Azure Blob → Event Grid → Autoloader → Bronze |
| **Ingestion Method** | NiFi (current) |
| **Bronze Target** | Endpoint security tables |

**Validation Focus:**
- Confirm Akamai URI registration (dependency D8 from Cribl Plan — currently not started)
- Validate WAF rules do not throttle or drop legitimate Tanium traffic at peak
- Test Autoloader performance at 68 TB/day sustained throughput
- Confirm Event Grid notification latency at high event rates
- Validate Parquet conversion and file sizing (target 128–512 MB)

### 3.3 WF2 — Data Center via NiFi (DC → NiFi → Blob → Databricks)

| Attribute | Detail |
|---|---|
| **Source** | Data center infrastructure and application logs |
| **Volume** | ~80 TB/day (DC workflow aggregate) |
| **Path** | DC sources → Apache NiFi (Cribl Worker) → Blob (public EP) → Databricks |
| **Bronze Target** | Multiple Bronze tables (infrastructure, VMware, Windows, databases) |

**Validation Focus:**
- **NiFi Disposition Assessment:** Evaluate NiFi retain / consolidate into Cribl / deprecate decision. This is a CVS architecture decision; KPMG provides assessment and recommendation considering:
  - PHI provenance: NiFi has built-in provenance logging; Cribl requires equivalent verification
  - Operational complexity: consolidating to Cribl reduces tool sprawl
  - Compliance evidence: determine which path provides stronger chain-of-custody evidence
- Validate DC connectivity through ExpressRoute
- Confirm Blob public endpoint access controls (IP allowlist + ACL/SP)

### 3.4 WF3 — Data Center via Splunk HF (DC → Splunk HF → Cribl Worker → Blob → DBX)

| Attribute | Detail |
|---|---|
| **Source** | DC sources currently routed through Splunk Heavy Forwarders |
| **Volume** | Varies by DC site |
| **Path** | DC → Splunk HF → Cribl Worker → Public Internet → Blob → Databricks |
| **Bronze Target** | Multiple Bronze tables |

**Validation Focus:**
- Validate HF-to-Cribl Worker cutover (parallel Splunk path stays live; rollback = config toggle)
- **Persistent Queue Sizing Risk:** Shea DC has <1 hour buffer capacity (HIGH risk). If downstream is unavailable for >1 hour, events may be lost. Validate queue sizing for all 6 DCs:
  - Atlanta, Vegas, Windsor, Middletown — adequate buffer
  - Arizona — moderate risk
  - Shea (RI) — **<1 hour buffer, requires remediation or acceptance**
- Confirm DC rollout order: Atlanta → Vegas → Windsor → Middletown → Arizona → Shea
- Validate private endpoint connectivity (10.100.9.4 for ADLS; 10.0.5.10 for Cribl Private Link)

### 3.5 WF4 — GCP (GCP → Cribl GKE → Azure Blob → DBX)

| Attribute | Detail |
|---|---|
| **Source** | GCP Pub/Sub, GCS, GKE audit logs |
| **Volume** | Moderate (part of 53.92 TB/day Cloud Platform total) |
| **Path** | GCP → Splunk HF → Cribl Worker (GKE) → Azure Blob → Databricks |
| **Bronze Target** | `bronze.gcp_pubsub` |

**Validation Focus:**
- Validate cross-cloud connectivity (GCP → Azure)
- Confirm Pub/Sub subscription configuration and delivery guarantees
- Test resource_type discriminator column for downstream Silver fan-out
- Validate Cribl Worker on GKE operational readiness

### 3.6 WF5 — AWS CrowdStrike (AWS → S3 → Cribl ECS → Azure Blob → DBX)

| Attribute | Detail |
|---|---|
| **Source** | CrowdStrike (endpoint detection) |
| **Volume** | ~16.35 TB/day (second largest single source) |
| **Path** | CrowdStrike → S3/Kinesis → SQS → Cribl Worker (ECS) → Azure Blob → Databricks |
| **Bronze Target** | Security endpoint tables |

**Validation Focus:**
- Validate CrowdStrike S3 export configuration and SQS notification reliability
- Test Cribl Worker on ECS at 16 TB/day sustained throughput
- Confirm cross-cloud latency (AWS → Azure) does not violate 5-minute SLA
- Validate OCSF-readiness of CrowdStrike events for downstream Silver normalization

### 3.7 WF6 — Azure Native (Azure → Event Hub/Blob → Cribl AKS → DBX)

| Attribute | Detail |
|---|---|
| **Source** | Azure-native logs (Activity Logs, Entra ID, Key Vault, NSG, Firewall, Storage) |
| **Volume** | Part of Cloud Platform total |
| **Path** | Azure Event Hub/Blob → Cribl Worker (AKS) → Databricks |
| **Bronze Target** | `bronze.azure_eventhub`, `bronze.azure_logs` |

**Validation Focus:**
- Validate Event Hub consumer group configuration and throughput units
- Confirm category column population for downstream Silver fan-out (AKS audit, Key Vault, App Service, PostgreSQL)
- Test Cribl Worker on AKS operational readiness
- Validate Z-Ordering on resource_provider for Azure resource analytics performance

---

## 4.0 Bronze Table Design and Mapping

### 4.1 Consolidation Logic

The current Splunk estate contains approximately 925 production indexes. Rather than creating a Delta table per index, telemetry is consolidated into ~25–50 governed Bronze tables based on logical source families defined by WS-B1 data contracts.

**Consolidation progression:**
```
925 Splunk Indexes
    → 126 Physical Source Families (from B1 taxonomy)
    → ~25–50 Bronze Delta Tables (governed by ODCS contracts)
```

### 4.2 Four Design Patterns

Each Bronze table follows one of four engineering patterns (from the Technical Implementation Approach):

**Pattern 1 — Azure Event Hub:**
Multiple Azure diagnostic categories stored in a single sourcetype → one Bronze table with `category` column as the Silver fan-out discriminator. Partition by `event_date` + `splunk_index`; Z-Order on `resource_provider`.

**Pattern 2 — GCP Pub/Sub:**
Multiple GCP Pub/Sub sourcetypes, indexes, and business units consolidated into one Bronze table. `resource_type` becomes the Silver discriminator. Labels and attributes retained as raw JSON until Silver normalization.

**Pattern 3 — Network Security (Multi-Vendor Consolidation):**
Vendor-specific telemetry (Cisco ASA, Palo Alto, F5, Check Point, Fortinet, Infoblox) ingested into a single `bronze.network_syslog` table with `vendor` as a column. Creates a unified enterprise network-security ingestion model while enabling vendor-neutral Silver datasets.

**Pattern 4 — Kubernetes Family Collapse:**
Hundreds of deployment-specific sourcetypes (e.g., `kube:container:crm-*`) consolidated into single Bronze tables where `deployment`, `namespace`, `cluster`, `pod`, and `environment` become dimension columns rather than schema identifiers. This is the largest simplification — 10,790 sourcetypes (84% of total) collapse into ~40 logical families.

### 4.3 Pattern Selection Criteria

| Source Characteristic | Recommended Pattern | Rationale |
|---|---|---|
| Single vendor, multiple diagnostic categories | Pattern 1 (Event Hub) | Category column enables Silver fan-out without duplicating Bronze |
| Multi-cloud, same data type | Pattern 2 (Pub/Sub) | Resource type discriminates; labels retained as raw JSON |
| Multi-vendor, same domain (e.g., firewalls) | Pattern 3 (Network Security) | Vendor column; common analytical fields standardized |
| High-cardinality deployment names (K8s) | Pattern 4 (Family Collapse) | Metadata promoted to columns; dramatically reduces table count |
| Unique domain, single vendor | Direct mapping | One source family → one Bronze table (e.g., `bronze.enterprise_health`) |

### 4.4 Canonical Bronze Schema

Every Bronze table conforms to a common enterprise metadata model regardless of source technology:

| Column | Data Type | Description |
|---|---|---|
| `ingest_timestamp` | Timestamp | Time telemetry entered Databricks |
| `source_timestamp` | Timestamp | Original event generation time |
| `bronze_product` | String | Bronze Data Product identifier |
| `source_family` | String | Logical source family (from B1 contract) |
| `splunk_index` | String | Original Splunk index |
| `splunk_sourcetype` | String | Original Splunk sourcetype |
| `application` | String | Business application or service |
| `business_domain` | String | Line of Business |
| `environment` | String | Production, QA, Development, etc. |
| `cloud_provider` | String | Azure, AWS, GCP, On-Premises |
| `region` | String | Geographic region or data center |
| `hostname` | String | Source host or system |
| `resource_id` | String | Cloud resource identifier |
| `resource_type` | String | Cloud resource classification |
| `severity` | String | Event severity where applicable |
| `raw_payload` | Variant/String | Original event payload (preserved unmodified) |
| `metadata` | Variant | Pipeline-generated metadata |

Technology-specific extensions (K8s: cluster, namespace, deployment, pod, container, node, labels, annotations; VMware: vcenter, cluster, host, VM, datastore; Cloud: subscription_id, resource_group, project, account; Network: vendor, device, interface, protocol, action, src/dst IP/port) are added per table type while maintaining the canonical envelope.

### 4.5 Partitioning Strategy

| Bronze Table Type | Primary Partition | Secondary Optimization |
|---|---|---|
| Cloud | `event_date` | Subscription / Resource Provider |
| Kubernetes | `event_date` | Cluster / Namespace |
| VMware | `event_date` | Cluster |
| Windows | `event_date` | Host |
| Network | `event_date` | Vendor |
| Security | `event_date` | Product |
| Database | `event_date` | Database Platform |
| Pharmacy | `event_date` | Application |
| Clinical | `event_date` | Application |

Additional optimization: Delta Auto Optimize, Auto Compaction, Liquid Clustering (where appropriate), Z-Ordering on frequently filtered dimensions, Predictive Optimization.

---

## 5.0 Autoloader and Event Grid Configuration

### 5.1 Why Event Grid Is Critical

At 400 TB/day, Autoloader cannot rely on recursive directory listing to discover new files — the volume and frequency would create unacceptable latency and compute cost. Event Grid provides push-based file notifications that trigger Autoloader ingestion immediately upon file arrival.

### 5.2 Autoloader Configuration

| Configuration Element | Approach |
|---|---|
| **Ingestion Mode** | File notification mode (Event Grid) — not directory listing |
| **Trigger** | Event Grid subscription on ADLS Gen2 container |
| **Checkpoint** | Managed checkpoint per Bronze table — enables exactly-once semantics and recovery |
| **Schema Handling** | Schema inference on first batch; additive schema evolution enabled for new fields; rescued data column for unexpected fields |
| **Processing** | Streaming for Tier 1 (hot) sources; triggered micro-batch for Tier 2/3 |
| **File Format** | Auto-detect JSON vs. Parquet based on landing path convention |
| **Error Handling** | Malformed files routed to quarantine path; alerts generated; do not block pipeline |

### 5.3 ADLS Path → Bronze Table Mapping

Each ADLS landing path maps deterministically to a single Bronze Delta table:

| ADLS Landing Folder | Bronze Delta Table |
|---|---|
| `/bronze/cloud/platform/` | `bronze.cloud_platform_telemetry` |
| `/bronze/security/network/` | `bronze.enterprise_network_security` |
| `/bronze/security/windows/` | `bronze.windows_endpoint_events` |
| `/bronze/infrastructure/vmware/` | `bronze.vmware_infrastructure` |
| `/bronze/pharmacy/rxconnect/` | `bronze.rxconnect_platform` |
| `/bronze/pharmacy/ivr/` | `bronze.rx_ivr_platform` |
| `/bronze/digital/kubernetes/` | `bronze.kubernetes_applications` |
| `/bronze/integration/` | `bronze.enterprise_integration` |
| `/bronze/clinical/enterprise_health/` | `bronze.enterprise_health` |
| `/bronze/audit/enterprise/` | `bronze.enterprise_audit` |

This deterministic mapping simplifies lineage, governance, operational monitoring, and downstream Silver transformations.

### 5.4 Streaming vs. Micro-Batch Decision

| Source Tier | Autoloader Mode | Trigger | Rationale |
|---|---|---|---|
| **Tier 1 — Hot** | Continuous streaming | Always-on | SIEM-feeding, real-time detection, SOC dashboards |
| **Tier 2 — Warm** | Triggered micro-batch | Every 15 min | Compliance, audit, operational reporting |
| **Tier 3 — Cold** | Scheduled batch | Daily | Historical, low-priority, long-tail sources |

This tiering reduces Databricks compute cost by 60–70% compared to running all sources in continuous mode.

---

## 6.0 Fork Routing Strategy

### 6.1 Routing Destinations

During the hybrid migration period, each source's telemetry may flow to one or more destinations:

| Routing Pattern | Description | When Used |
|---|---|---|
| **Databricks Only** | Source fully migrated; Splunk no longer receives data | Post-migration, Splunk decommissioning |
| **Splunk + Databricks** | Cribl forks to both; Splunk continues for operational use while Databricks builds analytics | Active migration period (most sources) |
| **Splunk + SIEM + Databricks** | Security sources feeding Chronicle/CrowdStrike AND Databricks | Security sources during SIEM evaluation |
| **Splunk Only** | Source not yet migrated to Databricks | Pre-migration (decreasing over time) |

**Chronicle Ingestion Scope:**
Google Chronicle currently ingests ~170 sources generating approximately 104 TB/day (dominated by Tanium Stream at ~68 TB and CrowdStrike at ~16 TB). Farzana's security team has already rebuilt much of the Chronicle detection capability in Databricks + AWS via a parallel path. Chronicle data ingestion into the Databricks Lakehouse may follow a separate pipeline (not routed through Cribl) depending on Farzana's existing architecture. B2's scope is to validate Cribl-routed telemetry landing in Bronze; Chronicle's ingestion path will be confirmed during the WS-A architecture assessment and may require a separate ingestion validation track if it bypasses Cribl.

### 6.2 Routing Decision Criteria

| Criterion | Decision Input |
|---|---|
| B1 contract status | Source must have approved data contract before Databricks routing activates |
| Silver pipeline readiness (WS-C) | Is there a Silver pipeline ready to consume this Bronze table? |
| Consumer readiness (WS-D/E) | Are migrated dashboards/queries ready for users? Are users trained? |
| Business criticality | Mission-critical sources (Pharmacy, SOC) get dual routing first; low-priority sources can wait |
| November freeze deadline | Any source needed before freeze must be dual-routed with sufficient validation time |

### 6.3 Rollback Capability

Parallel Splunk routing stays live during the entire migration. Rollback is a Cribl configuration toggle — no data loss, no infrastructure change. This provides a safety net for any Bronze landing issues.

---

## 7.0 Throughput Validation and SLA Testing

### 7.1 SLA Definition

**Primary SLA:** 99.9% of events ingested into Bronze Delta tables within 5 minutes of event generation.

**Measurement points:**
- `source_timestamp` — when the event was generated at the source
- `ingest_timestamp` — when the event was written to the Bronze Delta table
- Latency = `ingest_timestamp` − `source_timestamp`
- SLA met if P99.9 latency < 5 minutes over a 24-hour measurement window

### 7.2 Test Methodology

| Test Phase | Description | Duration | Success Criteria |
|---|---|---|---|
| **Smoke Test** | Inject 100 known events per workflow; confirm arrival in Bronze | 1 hour | 100% arrival, correct schema, correct table |
| **Volume Ramp** | Gradually increase to 50% of production volume per workflow | 4 hours | P99 latency < 5 min; no errors |
| **Production Parity** | Sustain at 100% production volume per workflow | 24 hours | P99.9 latency < 5 min; <0.1% error rate |
| **Burst Test** | Spike to 150% of production volume (+50% headroom) | 2 hours | P99 latency < 10 min; no data loss; recovery to SLA within 30 min |
| **Failure Recovery** | Simulate downstream outage (stop Autoloader); resume; validate replay | 1 hour | All buffered events replayed; no duplicates; checkpoint intact |

### 7.3 Per-Workflow Volume Benchmarks

| Workflow | Primary Source | Target TB/day | Burst Target (150%) |
|---|---|---|---|
| WF1 | Tanium | 68 TB | 102 TB |
| WF2 | DC / NiFi | ~80 TB (aggregate) | ~120 TB |
| WF3 | DC / Splunk HF | Varies by DC | +50% per site |
| WF4 | GCP Pub/Sub | Part of 53.9 TB cloud total | +50% |
| WF5 | CrowdStrike | 16.35 TB | 24.5 TB |
| WF6 | Azure native | Part of cloud total | +50% |

### 7.4 Small-File Mitigation

At high event rates, Cribl may produce many small files in ADLS. Small files degrade Autoloader and Delta query performance. Mitigation:

- **Cribl-side:** Configure target file size 128–512 MB via Cribl output buffering
- **Databricks-side:** Enable Delta Auto Compaction to merge small files automatically
- **Monitoring:** Alert if average file size drops below 64 MB for any Bronze table
- **Validation:** Confirm file size distribution during Production Parity test

### 7.5 Persistent Queue Validation

Cribl persistent queues buffer events during downstream outages. Queue capacity determines how long an outage can last without data loss.

| DC Site | Current Buffer | Risk Level | Validation Action |
|---|---|---|---|
| Atlanta | Adequate | Low | Confirm buffer > 4 hours |
| Vegas | ~4.4 hours | Low | Confirm buffer > 4 hours |
| Windsor | Adequate | Low | Confirm buffer > 4 hours |
| Middletown | Adequate | Low | Confirm buffer > 4 hours |
| Arizona | Moderate | Medium | Validate buffer duration; recommend increase if < 2 hours |
| **Shea (RI)** | **<1 hour** | **HIGH** | **Validate actual capacity; escalate to SciCom/CVS for remediation or risk acceptance** |

---

## 8.0 Pre-Freeze Readiness and Go-Live Criteria

### 8.1 November Freeze Context

CVS has a November production freeze that constrains when new platform changes can be deployed. All Tier 1 source data must be flowing into Bronze before the freeze window to ensure Workstreams C and D can begin operating against production data.

### 8.2 Tier 1 Activation Checklist

Before a source family is activated in production, the following must be confirmed:

| Checkpoint | Owner | Verified By |
|---|---|---|
| B1 data contract approved and registered in Unity Catalog | WS-B1 | Data Architect |
| Cribl routing pipeline configured and tested | SciCom | Cribl Lead |
| ADLS landing path created with correct permissions | SciCom / CVS | Platform Team |
| Autoloader configured with Event Grid notification | KPMG | Technical Lead |
| Bronze table created with canonical schema + partitioning | KPMG | Data Architect |
| Smoke test passed (100 events, correct schema, correct table) | KPMG | Data Engineer |
| Throughput test passed (production volume, 5-min SLA) | KPMG | SRE |
| Fork routing confirmed (Splunk parallel path active) | SciCom | Cribl Lead |
| Monitoring configured (ingestion lag, error rate, file size) | KPMG | SRE |
| Silver pipeline ready to consume (WS-C coordination) | WS-C | Data Architect |

### 8.3 Readiness Scorecard

Each source family is scored across five dimensions:

| Dimension | Green | Yellow | Red |
|---|---|---|---|
| **Connectivity** | End-to-end path validated | Path configured, not fully tested | Path not yet configured |
| **Throughput** | Production Parity test passed | Volume Ramp passed, Production Parity pending | Not yet tested |
| **Data Quality** | Schema validation passing; <0.1% error rate | Minor schema issues; <1% error rate | Schema failures; >1% error rate |
| **Governance** | Contract registered; table in Unity Catalog; lineage visible | Contract approved, registration pending | Contract not yet completed |
| **Monitoring** | Alerts configured; dashboards active | Alerts configured, dashboards pending | No monitoring in place |

**Go/No-Go Rule:** A source family is production-ready when all 5 dimensions are Green. Yellow requires documented risk acceptance. Any Red blocks activation.

### 8.4 Escalation Criteria

| Scenario | Action | Escalation Path |
|---|---|---|
| Source not ready 2 weeks before freeze | Escalate to PMO; assess whether source can be deferred to post-freeze wave | KPMG Program Lead → CVS PM |
| Persistent queue capacity insufficient | Escalate to SciCom for infrastructure remediation | KPMG Technical Lead → SciCom Cribl Lead → CVS Infrastructure |
| Throughput SLA consistently missed | Root cause analysis; engage Databricks support if Autoloader is bottleneck | KPMG SRE → Databricks account team |
| Fork routing conflicts with Splunk operations | Coordinate with CVS Splunk team to adjust timing | KPMG Splunk Lead → CVS Splunk/SIEM Team |

### 8.5 Coordination with Downstream Workstreams

| Workstream | B2 Provides | Timing |
|---|---|---|
| WS-C (Silver Pipelines) | Confirmed Bronze table schemas, partition keys, and data availability | C pipeline development begins once B2 confirms Tier 1 Bronze tables are populated |
| WS-D (Analytics Migration) | Production data in Bronze for query testing against Silver/Gold | D cannot validate migrated queries until B2 + C deliver queryable Silver tables |
| WS-E (OCM) | Evidence of data availability for user communications | E adoption messaging includes "your data is now in Databricks" milestones |
| WS-F (Hypercare) | Operational runbooks and monitoring baselines | F inherits B2's monitoring configuration and escalation procedures |

---

## 9.0 Alignment with SOW Scope and Deliverables

| SOW Deliverable | Methodology Section |
|---|---|
| Bronze Architecture and Data Landing Design | Section 2 (Architecture) + Section 4 (Table Design) |
| Ingestion Routing Matrix and Source-to-Bronze Mapping | Section 4.1 (Consolidation Logic) + Section 5.3 (Path-to-Table Mapping) |
| Cribl Routing Pipeline Configuration Documentation | Section 3 (Workflow Validation) — validation evidence per workflow |
| Autoloader and Event Grid Configuration Evidence | Section 5 (Autoloader and Event Grid) |
| Bronze Table/Partition Design Documentation | Section 4 (Table Design, Schema, Partitioning) |
| Throughput and SLA Validation Report | Section 7 (Throughput Testing) |
| Pre-Freeze Readiness Checkpoint and Handoff Materials | Section 8 (Readiness Scorecard, Activation Checklist) |
