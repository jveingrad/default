# CVS Data Architecture Approach: Workstreams A, B1/B2, C

**Lens:** data architecture, organized by the four Data Foundation areas in the Databricks Architecture Primer (Data Architecture, Data Ingestion and Processing, Storage Architecture, Medallion Architecture).
**Built from:** every file in the `CVS` folder (see Section 8). Where files disagree, Section 6 lists the conflict and a recommended ruling.
**Status:** working draft for SME review. Not client-ready until the Section 6 decisions are taken.

---

## 1. The integrated approach in one view

The four workstreams are one data-architecture chain. A proves and fixes the pattern, B1 decides which data is governed and how, B2 lands it in Bronze, C turns it into Silver and Gold products.

| From → To | What passes across | Gate to release |
|---|---|---|
| **A → B1** | Canonical metadata standard, ODCS template, Unity Catalog namespace hierarchy | Catalog taxonomy decision (Section 6, D2) |
| **A → B2** | Validated Auto Loader + Event Grid pattern, ADLS folder hierarchy, throughput benchmarks | Landing-zone and network clearance; A2 pilot performance baseline |
| **A → C** | Pilot as reference pattern: DLT code, Silver schema, RBAC and masking pattern, FinOps baseline | Pilot accepted by CVS product owner |
| **B1 → B2** | Approved contract per source family: routing rule, Bronze table, partition keys | Contract registered in Unity Catalog before the source activates |
| **B1 → C** | Schema, classification, SLA, masking rules, lineage, downstream specification | Approved contract for the target family |
| **B2 → C** | Populated Bronze tables, confirmed schemas and partition keys | Tier 1 Bronze tables carrying production data |
| **C → D** | Silver and Gold tables, including pre-aggregated tables for the 16 ES accelerated CIM data models | Queryable Silver data (hard sequencing dependency per WS-D §7.3) |

**Framework area × workstream** (L = leads, C = contributes, I = consumes):

| Framework area | A | B1 | B2 | C |
|---|---|---|---|---|
| Data Architecture (modeling, semantics, products) | L: sets pattern and decisions | L: contracts, taxonomy, classification | I | L: canonical model, conformed dimensions, data products |
| Data Ingestion and Processing | L: validates 1–3, scalability | C: routing rules | L: WF1–WF6, Auto Loader, tiering, SLA | L: DLT templates, hot/warm/cold tiers |
| Storage Architecture | L: ADLS, encryption, DR assessment | C: retention, classification | L: landing paths, file standards, queues | I |
| Medallion Architecture | L: pilot Bronze→Gold | C: contract = Bronze spec | L: Bronze | L: Silver and Gold |

---

## 2. Workstream A: Architecture Assessment, Pilot, Remediation (12 weeks)

**Data-architecture objective:** prove the Bronze → Silver → Gold pattern on real data and set the decisions B and C build on, so neither workstream reworks.

**Gap to close first.** A's seven assessment domains are platform-centric (Unity Catalog, security, encryption, DR, scalability, FinOps, CI/CD). Data modeling, Bronze design, landing-zone layout and ingestion patterns are only covered implicitly through the pilot. **Recommendation: add Domain 8, "Data Architecture and Medallion Design"** (and a matching feature 1.x in the kickoff epic), scored with the same Green/Yellow/Red rubric:

| Assessment item | Evidence | Green / Yellow / Red |
|---|---|---|
| Canonical Bronze metadata envelope defined and applied | Bronze DDL, schema registry | All tables carry it / partial / ad-hoc schemas |
| Source-family → Bronze-table consolidation documented | Mapping sheet, Bronze table list | Documented and ratified / draft / table per index |
| Landing-zone and folder convention, quarantine path | ADLS container inspection | By product/date with quarantine / convention without quarantine / ad hoc |
| Auto Loader configuration | Job config, checkpoint review | File notification, rescued-data column, quarantine / partly / directory listing with `mergeSchema` only |
| Partitioning, clustering, file size | Table details, file-size histogram | Average ≥128 MB / 64–128 MB / <64 MB (alert threshold in B2 §7.4) |
| Classification boundary enforced (Restricted → Internal) | UC tags, masking test evidence | Tested and tagged / defined, untested / undefined |
| Silver contract enforcement and quality expectations | DLT expectations, quarantine counts | Enforced from contract / manual / none |
| Retention by layer, WORM scope | Lifecycle rules, immutability policy | Active and tested / defined only / missing |

**Approach**

1. **Weeks 1–3, inventory.** Catalogs, schemas, tables, Auto Loader jobs, ADLS containers and lifecycle rules across the ~201 workspaces. Treat the departed architect's artifacts as undocumented until proven. Coordinate with Lovelytics so security domains are not assessed twice.
2. **Weeks 4–6, assess and score.** Run the eight-domain checklists. Test the PHI masking boundary with edge cases (schema evolution, late-arriving fields). Load-test Auto Loader at representative volume. Gap register v1 in Week 4, report in Week 6.
3. **Weeks 7–10, pilot (K8s/CRM or network security).** Build the pilot as a reference: ODCS contract, Bronze table with envelope, parameterized DLT pipeline with masking and expectations, one enrichment join, Gold output, concurrency and latency tests.
4. **Weeks 4–12, remediate and decide.** Must-fix items before the change freeze; take the Section 6 decisions; assemble the B–E handoff package.

**Outputs:** assessment report with data-architecture domain, gap register, pilot evidence, remediation tracker, Day 1 / Day 2 architecture, handoff package. Dates in the kickoff deck: A.1 definition of done Dec 7, 2026; A.2 (90-day window) Jan 18, 2027.

**Exit gate:** B1, B2 and C can start from the handoff without rework, and the Section 6 decisions D1–D4 are closed.

---

## 3. Workstream B1: Data Contracts and Source Rationalization

**Data-architecture objective:** turn 12,910 sourcetypes into ~250–300 governed source families whose contracts *are* the Bronze and Silver specifications.

**Approach (four-step funnel, factory model)**

1. **Discover and group** (AI-assisted pattern recognition): 12,910 sourcetypes → ~250–300 families → 12 source-domain categories. Kubernetes (10,790 sourcetypes, 84%) collapses to ~40 families by promoting deployment, namespace, cluster and environment to columns.
2. **Classify and tier:** Tier 1 = ~126 production-validated families (145.4 TB/day, ~98% of volume), contracted in Weeks 1–10; Tiers 2 and 3 follow by Week 20.
3. **Disposition:** Retain & Contract / Consolidate / Retire, informed by usage of dependent searches, dashboards and knowledge objects.
4. **Author and register:** AI pre-populates ODCS contracts; owner workshop; register in Unity Catalog.

**Data-architecture checks the SME applies to every contract before release**

- The contract names exactly one Bronze target table, partition keys and the canonical envelope columns (consistent with B2 §4.4).
- Family-specific extension columns follow the B2 patterns (Event Hub, Pub/Sub, network multi-vendor, K8s collapse).
- Classification is the highest across consuming platform domains (Core, ES, ITSI); masking layer is stated explicitly (see D1).
- Downstream Silver tables, Tier 1/2 normalization (OCSF vs governed native) and tier (hot/warm/cold) are declared so C does not re-derive them.
- Retention by layer (contract example: Bronze 13 months, Silver 13 months, 7-year WORM archive) matches storage lifecycle design.

**Constraint to state openly:** KPMG authoring capacity (15–20 contracts per sprint) is not the bottleneck; CVS data-owner availability is. Ownership is concentrated (top 25 owners hold >90% of data), so domain-grouped workshops are the lever.

**Outputs:** source inventory and taxonomy, contract template and standards, completed contracts, classification and security boundary matrix, Unity Catalog registration standards, BAU handoff.
**Exit gate:** all Tier 1 contracts registered by Week 10 (pre-freeze checkpoint).

---

## 4. Workstream B2: Bronze Data Landing

**Data-architecture objective:** every ingestion workflow lands reliably in governed Bronze Delta tables within the 5-minute SLA, with the design KPMG validates (SciCom executes Cribl and infrastructure).

**Approach**

1. **Design Bronze (Data Architecture + Medallion).** Consolidate 925 indexes → 126 families → ~25–50 Bronze tables using the four patterns; canonical envelope on every table; partition by `event_date` with a secondary dimension per table type.
2. **Fix the storage contract (Storage).** Landing paths by Bronze product, `Product/Year/Month/Day/Hour`, JSON or Parquet, 128–512 MB files, CMK encryption, lifecycle and optional WORM; deterministic path → table mapping.
3. **Configure ingestion (Ingestion).** Auto Loader in file-notification mode via Event Grid, managed checkpoints, additive schema evolution with rescued-data column, malformed files to quarantine; streaming for Tier 1, 15-minute micro-batch for Tier 2, daily batch for Tier 3 (60–70% compute saving).
4. **Validate each workflow** with the five-step sequence (connectivity, sample, schema, throughput, 24-hour SLA), including persistent-queue sizing per data center (Shea <1 hour is the known high risk) and the NiFi disposition and PHI chain-of-custody assessment.
5. **Test throughput:** smoke, ramp, production parity, 150% burst, failure recovery; measure `ingest_timestamp − source_timestamp`.
6. **Gate activation** with the readiness scorecard (connectivity, throughput, data quality, governance, monitoring): all Green, Yellow only with risk acceptance, any Red blocks.

**Outputs:** Bronze design, routing matrix, Autoloader/Event Grid evidence, table and partition design, throughput and SLA report, pre-freeze readiness checkpoint.
**Exit gate:** Tier 1 sources flowing with scorecard Green before the freeze; C can read populated Bronze.

---

## 5. Workstream C: Data Architecture and Pipeline Enablement

**Data-architecture objective:** deliver the canonical model, template-driven Silver, governed data products and Gold self-service.

**Approach**

1. **Canonical model (Weeks 1–3).** Three tiers: Tier 1 OCSF/ECS for security data needing cross-domain detection; Tier 2 governed native schema for observability data; Tier 3 common metadata envelope on all tables. Document source → model → ontology mapping (answers the Jul 28 request).
2. **Conformed dimensions first.** Specify the 15 dimensions (SCD Type 2 for application, asset, business service, configuration item, business unit, cloud resources). Silver enrichment depends on CMDB-quality `dim_asset` and `dim_application`, so these land before most domain pipelines.
3. **Template library (Weeks 4–10).** 18–22 parameterized DLT templates across eight categories, seeded by the A pilot. Each template: validate schema → parse → normalize → enrich → mask → de-duplicate → quality expectations. Tier hot/warm/cold from the contract SLA.
4. **Silver domains (Weeks 7–18).** Nine domains over four pods; many instantiations (240–320) converge on fewer physical tables (68–150). Engagement capacity supports ~350–400 priority pipelines of ~1,000 total.
5. **Governance.** Bronze Restricted → Silver Internal via regex, NER and tokenization masking; Unity Catalog tags, row filters, column masks; HIPAA evidence (access, masking, classification, lineage, retention).
6. **Data products and Gold (Weeks 11–22).** Promote Silver tables with a named consumer, product owner, published SLOs and >99% expectation pass rate; 12 Gold products; Genie semantic models; team-owned Gold workspaces that read Silver, never Bronze.

**Outputs:** canonical model and mapping, template library, pipeline inventory, Silver data model and product catalog, conformed dimensions, UC governance configuration, Gold workspace design, runbooks.
**Exit gate:** Silver CIM-domain tables queryable for D; Gold workspaces ready for E's adoption waves.

---

## 6. Decisions and conflicts the SME needs to rule on

| # | Issue | Where it conflicts | Recommended ruling |
|---|---|---|---|
| **D1** | **Where PHI/PII is masked and what Bronze contains.** Walkthrough §5 says masking happens at Cribl before the lake, yet §6/§11 treat Bronze as Restricted with unmasked PHI. B1 contract example sets `cribl_guard_pre_ingestion` (Jul 31). C masks in DLT at Silver. | Walkthrough §5/§6, B1 §3.2, C §5.1 | Defense in depth: Cribl Guard for known fields, DLT masking at Silver, Bronze stays Restricted until masking is proven by the OI-006 test. Tie to OI-002 (NiFi vs Cribl). |
| **D2** | **Catalog structure.** Walkthrough proposes `observability_prod`, `security_prod`, `shared_enrichment`, `workspaces_user`; C says one catalog per data domain (~8–12). | Walkthrough §11, C §5.2 | A decides in A3 and hands B1 the namespace; consider domain × environment with a shared enrichment catalog. |
| **D3** | **Table counts.** Bronze: ~25 (walkthrough, B1) vs 25–50 (B2) vs "26-50" (kickoff). Silver: 20–30 (walkthrough) vs 68–150 physical / 240–320 instantiations (C). | Several | Use "~25–50 Bronze" and C's reconciliation (instantiations → physical tables → 9 domains → 12 Gold); get CVS architecture-lead sign-off. |
| **D4** | **Workflow numbering.** Walkthrough: WF1 = data center primary path. B2: WF1 = Tanium via Akamai, WF2 = data center via NiFi. SAIPA documents: WF1 DC, WF2 multi-cloud, WF3 Tanium. | Walkthrough §5, B2 §3, Appendix B | Publish one workflow registry before any test evidence is filed. |
| **D5** | **Bronze table naming.** `bronze.network_syslog` (B1 contract, B2 Pattern 3) vs `bronze.enterprise_network_security` (B2 path map). | B1, B2 | Pick one; the path-to-table map should be the source of truth. |
| **D6** | **Latency SLA.** 99.9% in 5 minutes (B2), OI-003 asks <30 s vs 5 min, diagram note says <10 s, Event Grid green threshold <30 s (A). | Walkthrough, A, B2 | Define the clock (source to Bronze write) per tier; keep 5 minutes for hot as the contractual SLA. |
| **D7** | **Freeze date.** B1/B2 say November freeze; kickoff says December 1 change freeze. | B1/B2 vs kickoff | Confirm with CVS PMO. |
| **D8** | **Chronicle path.** Chronicle (~104 TB/day) may bypass Cribl. | B2 §6.1 | Decide in A whether a separate ingestion validation track is needed. |
| **D9** | **Document currency.** The `.md` methodologies for B1 and C are newer than the `_DRAFT.docx` copies (the docx lacks the Cribl Guard confirmation and C's table-count reconciliation). | CVS folder | Regenerate the docx files from the `.md` before circulating. |

---

## 7. SME changes made to the primer (A_DB Architecture Assessment Primer)

File: `A_DB Architecture Assessment Primer - SME Refined.pptx` (original left untouched).

| Slide | Change | Why |
|---|---|---|
| 2 Data Foundation | Rebuilt all four quadrants; widened right-hand boxes into unused slide space; real bullets; explicit heading formatting | The original used only the left two-thirds of the slide and left Medallion nearly empty |
| 2 Data Architecture | Single "canonical data model" bullet became a tiered model (OCSF/ECS, governed native, common envelope); added conformed dimensions, source → model → ontology, ODCS contract fields, product SLOs and promotion criteria | OCSF is security-only (Robinson, Jun 25), so one canonical model does not fit observability |
| 2 Ingestion | Added Event Grid file notification, latency tiers, idempotency, late-arriving data, schema evolution with rescued-data column, quarantine, and Cribl upstream | These are the failure modes at scale; ADF alone does not describe telemetry ingestion |
| 2 Storage | Landing zone now Incoming / Validated / Quarantine; folders by product not source index; added ADLS Access (missing from slide 2 but on slide 1); file-size target; cold-copy caveat on Archive tier | Archive-tier blobs cannot be read by Delta without rehydration, so lifecycle tiering must not touch active tables |
| 2 Medallion | Added the three layers' contracts of responsibility, classification per layer, Gold guardrail (no Bronze access), and a Delta Lake row | Slide 1 listed Delta Lake but slide 2 omitted it |
| 1 | Removed stray periods ("Delta Lake.", "Processing Controls.") | Sub-heading names on slide 1 now match slide 2 exactly |
| 4 | Replaced the duplicated business question (reconciliation tolerances appeared twice) with a data-definition question | Clear defect |
| 5 | Added interview role 7, Ingestion / Pipeline Engineer | Nobody on the list owned Cribl/NiFi/Event Grid/Auto Loader configuration |

**Not changed (flag for the next pass):** slides 4 and 5 are Splunk-specific while slides 1–3 are generic framework content; on slide 4 the top tech-oriented text slightly overlaps its header banner (pre-existing); on slide 1 the centre hexagon label wraps in LibreOffice rendering, so check it in PowerPoint.

---

## 8. Sources

`Kickoff Deck DRAFT.pptx`; `A_DB Architecture Assessment Primer.pptx`; `workstream-A_architect-environment-walkthrough (1).md`; `workstream-A_architecture-assessment-methodology_DRAFT.docx`; `workstream-B1_data-contracts-methodology.md` and `_DRAFT.docx`; `workstream-B2_bronze-data-landing-methodology.md` and `_DRAFT.docx`; `workstream-C_data-architecture-methodology.md` and `_DRAFT.docx`; `workstream-D_observability-rationalization-methodology_DRAFT.docx` (used for the C → D dependency and Chronicle scope).
