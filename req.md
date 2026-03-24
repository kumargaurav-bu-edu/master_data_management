# MDM Solution Requirements — Customer Data

## 1. Executive Summary

This document proposes a 3-phase Master Data Management (MDM) solution for customer account data built on top of the existing Salesforce + DNB architecture. The solution migrates legacy search and storage infrastructure to a modern AI-powered stack, introduces intelligent enrichment, and delivers a golden-record master with full lifecycle governance. The architecture is designed for extensibility, with a future SAP ERP integration path documented across all phases.

| Phase | Focus | Key Outcome |
|-------|-------|-------------|
| Phase 1 | Data Ingest & Migration | Modern ingest pipeline, Vector DB + AI search, PANW Customer ID |
| Phase 2 | Data Enrichment | AI-driven enrichment (Gemini Pro), hierarchy inference, governance UI |
| Phase 3 | Data Mastering | Deduplication, survivorship, golden record, relationship graph |

---

## 2. Current State & Challenges

### 2.1 Write-Side Challenges

| ID | Challenge | Impact |
|----|-----------|--------|
| WC-01 | Accounts created via Salesforce forms lack D-U-N-S at creation time, so ultimate parent (apex) and subsidiary relationships cannot be determined. | Orphaned records with no hierarchy context. |
| WC-02 | API-created accounts contain incomplete attributes; downstream enrichment runs only on a yearly cadence. | Long periods of low-quality or missing data. |
| WC-03 | No governance or validation framework controls account creation, allowing duplicates and inconsistencies from both manual and API paths. | Proliferating duplicate and conflicting records. |
| WC-04 | Vendor data used for enrichment has very low website coverage (~9%), limiting website-based enrichment and matching. | Unreliable domain-based matching and segmentation. |
| WC-05 | Informatica enrichment has limited data recovery — null inputs (e.g., street address) often produce null outputs instead of populating valid values. | Enrichment fails silently; data gaps persist. |
| WC-06 | No real-time or near-real-time duplicate detection during account creation. | Late discovery of duplicates and costly downstream remediation. |
| WC-07 | No standardized matching and survivorship policy (which source should win). | Inconsistent master records across systems. |
| WC-08 | Account hierarchy changes (merges, acquisitions, re-parenting) are not tracked or audited. | No historical relationship context. |
| WC-09 | Data quality issues are identified post-creation rather than prevented via pre-creation validation. | Reactive rather than proactive quality management. |
| WC-10 | Limited visibility and reporting on account creation metrics (duplicate rate, enrichment success rate, missing critical attributes). | Cannot measure or improve data quality. |
| WC-11 | No sellable entity classification at account creation. DNB and Zoom provide a `sellable_entity` flag indicating the company has decision-making ability, but this signal is not used to distinguish customers from locations during ingest. | Non-sellable locations created as full accounts, polluting the customer master and inflating account counts. |

### 2.2 Read-Side Challenges

| ID | Challenge | Impact |
|----|-----------|--------|
| RC-01 | Poor domain data quality limits Marketing's ability to enrich or segment accounts using website domains. | Marketing campaigns under-target or mis-target. |
| RC-02 | CyberArch matching struggles with similar/overlapping account names due to lack of a match classification and ranking framework. | Incorrect candidate selection; manual workarounds. |
| RC-03 | Search does not support subsidiary discovery when direct company name is unknown — relies on semantic name matching, not hierarchy signals. | Sales and compliance cannot navigate org structures. |
| RC-04 | No PANW internal persistent hierarchy identifier; vendor-provided IDs are used to derive parent/apex. Records without vendor IDs cannot be placed in hierarchy. | Broken hierarchy for vendor-ID-less records. |
| RC-05 | Search and match results are not explainable — no match rationale or contributing attributes exposed. | Low user confidence; high manual verification effort. |
| RC-06 | No unified view combining name, domain, address, and hierarchy signals during read-time matching. | Fragmented and inconsistent search outcomes. |
| RC-07 | No relationship-aware search (e.g., search by parent, discover all subsidiaries). | Limited usability for sales, marketing, compliance. |
| RC-08 | Limited monitoring of search effectiveness and match accuracy (false positives, false negatives, user re-selections). | Cannot continuously improve matching logic. |

---

## 3. Functional Requirements

### 3.1 Phase 1 — Data Ingest & Migration

| ID | Requirement | Challenge Addressed | Priority |
|----|-------------|---------------------|----------|
| FR-101 | Replace Elasticsearch-based search with FSAII AI vector search for semantic matching across name, domain, address, and phone attributes. | RC-01, RC-02, RC-05 | P0 |
| FR-102 | Migrate structured analytics storage from legacy BigQuery tables to a new partitioned schema (`staged_accounts`) in BigQuery, with CDC sync to Google Vector DB. | RC-06 | P0 |
| FR-103 | Onboard Zoom as a third source system alongside Salesforce and DNB, with schema harmonization to `cssot_` prefix convention. | WC-04 | P0 |
| FR-104 | Generate a `panw_customer_id` (UUID or deterministic hash) for every ingested record at staging time, creating the persistent internal identifier. | RC-04, WC-06 | P0 |
| FR-105 | Implement real-time dedup pre-check during ingest using composite signature (name + website + country + sellable_entity_id) with configurable threshold (default 0.75). | WC-06, WC-03 | P0 |
| FR-106 | Standardize all incoming fields: snake_case, date normalization, numeric phone formatting, domain lowercasing and deduplication. | WC-02 | P1 |
| FR-107 | Capture source provenance map and lineage fields for every record (source system, ingest timestamp, transformation log). | WC-09, WC-10 | P1 |
| FR-108 | Route possible-duplicate records to a review queue with confidence scores for steward triage. | WC-03, WC-06 | P1 |
| FR-109 | Emit audit events for every ingest action (create, merge-candidate, quarantine) to an immutable audit log. | WC-08, WC-09 | P1 |
| FR-110 | Support dual-run (shadow mode) of old Elastic search and new FSAII search for validation before cutover. | — | P1 |
| FR-113 | **Sellable Entity Classification**: After cross-source merge, check DNB/Zoom `sellable_entity` flag to classify each record. If sellable=true or no DNB/Zoom entry exists, classify as `customer` (eligible for golden record). If sellable=false and source is Salesforce, classify as `location` (not eligible for golden). SAP records retain their own account type classification regardless of sellable flag. | WC-11, WC-03 | P0 |
| FR-114 | **Location Record Handling**: Records classified as `location` bypass dedup pre-check, are staged with `record_class: location`, and linked to a parent customer entity via `parent_panw_id`. Location records are persisted but never promoted to golden records. | WC-11 | P0 |
| FR-111 | **(Future) SAP Integration**: Onboard SAP ERP as a source system with account type mapping (ZEND, ZSLD, ZSHP, ZBIL, ZPYR, ZPRS) to normalized `cssot_account_type` and `cssot_role` fields. Resolve multiple SAP partner roles for the same legal entity to a single `panw_customer_id`. | WC-03, WC-06 | P2 |
| FR-112 | **(Future) SAP Account Type Mapper**: Map SAP `account_group` codes to MDM-normalized account types; preserve original `sap_partner_function` for traceability. Maintain `account_roles` table linking partner functions to entities. | WC-07 | P2 |

### 3.2 Phase 2 — Data Enrichment

| ID | Requirement | Challenge Addressed | Priority |
|----|-------------|---------------------|----------|
| FR-201 | Integrate Gemini Pro for AI-driven attribute enrichment (fill missing fields: DUNS, industry, company type, employee range) with confidence scoring. | WC-05, WC-02 | P0 |
| FR-202 | Infer `ultimate_parent` and `direct_parent` hierarchy relationships using AI + vendor reference graph. | WC-01, RC-03, RC-04 | P0 |
| FR-203 | Build a Governance UI where data stewards can review enrichment changes, approve or reject, with change diffs. | WC-03, WC-09 | P0 |
| FR-204 | On steward approval, auto-sync enriched fields to Salesforce via API and to downstream consumers. | WC-02 | P0 |
| FR-205 | Handle late-arriving enrichment data (e.g., Zoom data arriving months later) by triggering re-enrichment and updating Vector DB + BigQuery. | WC-02, WC-04 | P0 |
| FR-206 | Assign `duplicate_risk_score`, `enrich_confidence`, and `match_rationale` to every enriched record. | RC-05, RC-08 | P1 |
| FR-207 | Validate key enrichment attributes against quality rules (website confidence > 80%, address match, DUNS verification). | WC-05 | P1 |
| FR-208 | Route rejected records to quarantine with feedback reason; track `enrich_success_rate`, `duplicate_rate`, `approval_latency`. | WC-10, RC-08 | P1 |
| FR-209 | Version all enrichment changes using SCD Type 2 in BigQuery for rollback and point-in-time queries. | WC-08 | P2 |
| FR-212 | **Sellable Entity Reclassification**: When late-arriving DNB/Zoom data changes the `sellable_entity` flag, trigger reclassification. A customer can be downgraded to location (if DNB now says not sellable, SF source) or a location can be upgraded to customer (if DNB/Zoom now says sellable). Reclassification requires steward approval and triggers unlink/link from golden record in Phase 3. SAP-sourced records are exempt — SAP classification always takes precedence. | WC-11, WC-02 | P0 |
| FR-210 | **(Future) SAP Hierarchy Cross-Validation**: Use SAP customer hierarchy data to validate or confirm AI-inferred hierarchy (upgrade `hierarchy_status` from "inferred" to "confirmed" when SAP and Gemini Pro agree). | WC-01, RC-04 | P2 |
| FR-211 | **(Future) SAP Bi-Directional Sync**: On steward approval, auto-sync enriched fields back to SAP master data via IDoc / BAPI / OData alongside Salesforce sync. | WC-02 | P2 |

### 3.3 Phase 3 — Data Mastering

| ID | Requirement | Challenge Addressed | Priority |
|----|-------------|---------------------|----------|
| FR-301 | Implement deduplication engine using FSAII vector similarity + deterministic rules across email, website, name, DUNS, phone, country, and hierarchy signals. | WC-06, WC-07 | P0 |
| FR-302 | Apply configurable survivorship policy with source preference order (Salesforce > SAP > DNB > Zoom > External) and field-level rules (latest non-null, highest confidence, trust score). | WC-07 | P0 |
| FR-303 | Generate `panw_golden_id` and maintain equivalence sets linking all contributing `panw_customer_id` values and `source_id` values. | RC-04 | P0 |
| FR-304 | Build a Golden ID Graph with parent/subsidiary relationships, supporting relationship-aware read APIs (parent lookup, subsidiary discovery, explain). | RC-03, RC-07 | P0 |
| FR-305 | Persist golden records in an SCD Type 2 master store with `orchestration_status`, `survivor_policy`, `survivor_metadata`, and `audit_ts`. | WC-08 | P0 |
| FR-306 | Publish `customer.golden.record` events to Kafka/event bus for downstream consumer distribution. | — | P0 |
| FR-313 | **Sellable Entity Gate for Golden Record**: Only records with `record_class: customer` or SAP-typed records are eligible for golden record creation. Records classified as `location` are stored with `master_status: location`, linked to a parent golden record via `location_of` edge in the Golden ID Graph, and are never promoted to golden records. | WC-11 | P0 |
| FR-307 | Track hierarchy events (parent_change, apex_change, acquisition, split) with full before/after audit. | WC-08 | P1 |
| FR-308 | Emit explainable match results with `match_rationale`, `matched_fields`, and `score` for every dedup decision. | RC-05 | P1 |
| FR-309 | Provide CyberArch-specific match API that returns ranked candidates with classification scores and hierarchy context. | RC-02 | P1 |
| FR-310 | Maintain match effectiveness dashboard: duplicate rate, false positive rate, false negative rate, user re-selection rate. | RC-08, WC-10 | P2 |
| FR-311 | **(Future) SAP Multi-Role Dedup Awareness**: Dedup engine must recognize that multiple SAP partner records (Sold-To, Ship-To, Bill-To, Payer) for the same legal entity are roles, not duplicates. Preserve all partner roles on the golden record via `golden_partner_roles` table. | WC-06, WC-07 | P2 |
| FR-312 | **(Future) Role-Based Graph Edges**: Extend the Golden ID Graph with SAP partner role edges (sold_to, ship_to, bill_to, payer) alongside parent/subsidiary hierarchy edges, enabling role-based traversal queries. | RC-03, RC-07 | P2 |

---

## 4. Non-Functional Requirements

### 4.1 Performance & Scale

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-01 | Ingest throughput (Phase 1) | >= 500 records/second sustained; burst to 2,000/sec |
| NFR-02 | Real-time dedup pre-check latency | < 500ms per record (p99) |
| NFR-03 | Vector search query latency (FSAII) | < 200ms per query (p95) |
| NFR-04 | AI enrichment latency (Gemini Pro, Phase 2) | < 5 seconds per record (p95) |
| NFR-05 | Salesforce auto-sync latency (post-approval) | < 30 seconds |
| NFR-06 | Golden record resolution latency (Phase 3) | < 1 second per merge decision (p95) |
| NFR-07 | Expected account volume | ~2M existing accounts; ~10K new accounts/month (confirm with business) |
| NFR-08 | Vector DB storage projection | ~2M records x 768-dim embeddings + metadata ≈ 15–20 GB (review with vendor) |
| NFR-09 | BigQuery storage projection | Partitioned by ingest_ts; estimated 50–100 GB across all phase tables (review annually) |

### 4.2 Availability & Reliability

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-10 | Ingest pipeline availability | 99.9% uptime |
| NFR-11 | Search / match API availability | 99.95% uptime |
| NFR-12 | Governance UI availability | 99.5% uptime (business hours) |
| NFR-13 | Data freshness — Vector DB to BigQuery CDC lag | < 15 minutes |
| NFR-14 | Recovery Point Objective (RPO) | < 1 hour |
| NFR-15 | Recovery Time Objective (RTO) | < 4 hours |

### 4.3 Security & Compliance

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-16 | PII classification: customer names, addresses, phone numbers, emails must be classified as PII and handled accordingly. | Mandatory |
| NFR-17 | Encryption at rest for Vector DB and BigQuery stores (AES-256 or equivalent). | Mandatory |
| NFR-18 | Encryption in transit (TLS 1.2+) for all API calls, CDC streams, and Salesforce sync. | Mandatory |
| NFR-19 | Role-based access control (RBAC) on Governance UI: Admin, Data Steward, Viewer roles with segregated permissions. | Mandatory |
| NFR-20 | Audit logs must be immutable (append-only) and retained for a minimum of 7 years. | Mandatory |
| NFR-21 | GDPR / CCPA compliance: support data subject access requests (DSAR) and right-to-erasure across all stores (Vector DB, BigQuery, golden master, audit). | Mandatory |
| NFR-22 | Data residency: confirm whether customer data must remain in specific geographic regions; configure Vector DB and BigQuery regions accordingly. | TBD — confirm with legal |
| NFR-23 | PII must not be stored in plain text inside vector embeddings; use hashed or tokenized representations where embeddings could be reverse-engineered. | Mandatory |

---

## 5. Technology Decisions

### 5.1 FSAII — AI Vector Search

FSAII (Few-Shot Adaptive Inference Indexing) is the AI-powered search technique replacing Elasticsearch. It generates contextual 768-dimensional vector embeddings for text attributes (name, domain, address, phone), enabling:

- **Semantic similarity search**: "Acme Corp" matches "A.C.M.E. Corporation" via meaning, not string overlap.
- **Fuzzy and synonym-aware matching**: reduces false negatives that plague keyword-based inverted indexes.
- **Explainable results**: each match returns contributing attributes and confidence scores.
- **Expected improvement**: duplicate detection accuracy from ~70% (Elastic) to ~90% (FSAII), based on benchmarks with domain + address + hierarchy signals in vector space.

### 5.2 Google Vector DB

Operational search engine for AI-powered matching and deduplication. Stores 768-dim embeddings with JSON metadata (panw_customer_id, source_system, quality_scores). Cosine similarity indexing with metadata filtering.

### 5.3 BigQuery

Structured analytics and reporting engine. Partitioned tables for staged, enriched, and golden records. SQL-accessible for ETL, dashboards, and compliance reporting. Source of truth for non-vector data.

### 5.4 Gemini Pro

Google's large language model used in Phase 2 for attribute enrichment, hierarchy inference, and field prediction. Operates with confidence scoring and fallback to rule-based enrichment when AI confidence is below threshold.

### 5.5 Kafka / Event Bus

Event-driven distribution layer for Phase 3 golden record publishing and CDC between Vector DB and BigQuery.

---

## 6. Data Sources

| Source | Type | Records Provided | Phase Onboarded |
|--------|------|------------------|-----------------|
| Salesforce | Customer Input System | Account records (name, website, address, phone, region) | Phase 1 (existing) |
| DNB (Dun & Bradstreet) | Vendor Reference Data | Company profiles (DUNS, company type, industry, hierarchy) | Phase 1 (existing) |
| Zoom | Vendor Enrichment Data | Firmographic data (company name, zoom_id, firmographics, industry) | Phase 1 (new) |
| SAP ERP *(future)* | Transactional / ERP System | Customer master records with multi-role partner functions (End Customer, Sold-To, Ship-To, Bill-To, Payer, Prospect), customer hierarchy, credit data, payment terms, sales org assignments | Future (dotted integration across all phases) |

---

## 7. Identity Model

```
source_id (e.g., SF-555)
    │
    ▼  [Phase 1: assigned at ingest]
panw_customer_id (e.g., PANW-0000-123)
    │
    ▼  [Phase 3: assigned at mastering]
panw_golden_id (e.g., PANW-GOLD-000001)
```

- **source_id**: Original identifier from the source system. Immutable.
- **panw_customer_id**: Internal staging identifier generated in Phase 1. Links a record through enrichment. One per ingested record.
- **panw_golden_id**: Master identifier generated in Phase 3. Represents the single golden entity. One per deduplicated customer. Maps to one-or-many `panw_customer_id` values via equivalence set.

---

## 8. Success Metrics & Baselines

| Metric | Current Baseline (Estimated) | Phase 1 Target | Phase 2 Target | Phase 3 Target |
|--------|------------------------------|----------------|----------------|----------------|
| Duplicate detection accuracy | ~70% | >= 85% | >= 90% | >= 95% |
| Search precision (top-5 results) | ~60% | >= 80% | >= 85% | >= 90% |
| Enrichment success rate (non-null fill) | ~40% (Informatica) | — | >= 80% | >= 85% |
| Website domain coverage | ~9% | >= 20% (Zoom adds) | >= 50% | >= 60% |
| Hierarchy coverage (parent assigned) | ~30% | >= 40% | >= 70% | >= 85% |
| Avg. time to detect duplicate | Days to weeks | < 1 minute | < 1 minute | < 1 minute |
| Avg. enrichment cycle time | Yearly | — | < 24 hours | < 24 hours |
| False positive rate (matching) | Unknown | Establish baseline | < 10% | < 5% |
| False negative rate (matching) | Unknown | Establish baseline | < 15% | < 8% |
| Governance approval latency | N/A | — | < 48 hours (p90) | < 48 hours (p90) |
| Sellable entity classification accuracy | Unknown | >= 90% | >= 95% | >= 95% |
| Location-to-customer reclassification rate | N/A | Establish baseline | < 5% of locations reclassified | < 3% |

> **Note**: Baseline numbers are estimates and should be validated with current production data before Phase 1 begins.

---

## 9. Phased Delivery Timeline (Indicative)

| Phase | Duration (Estimated) | Key Milestones |
|-------|---------------------|----------------|
| Phase 1 | 3–4 months | Ingest pipeline live, Vector DB + BigQuery populated, PANW Customer ID assigned, Zoom onboarded, shadow-mode search running |
| Phase 2 | 3–4 months | Gemini Pro enrichment live, hierarchy inference active, Governance UI deployed, Salesforce auto-sync enabled |
| Phase 3 | 4–5 months | Dedup engine live, golden records published, relationship graph available, Kafka distribution active, compliance dashboards deployed |

---

## 10. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| FSAII search quality does not meet accuracy targets | Medium | High | Run shadow mode (FR-110) comparing FSAII vs. Elastic; tune embeddings before cutover |
| Gemini Pro availability or rate limits constrain enrichment throughput | Medium | High | Implement fallback to rule-based enrichment; queue and retry with exponential backoff |
| Vector DB and BigQuery drift out of sync | Medium | Medium | CDC with < 15-minute SLA (NFR-13); reconciliation job runs daily; alert on drift > threshold |
| Salesforce API sync failures during batch enrichment | Medium | Medium | Retry with dead-letter queue; steward notification on persistent failure; partial-success handling |
| Governance review queue backlog stalls enrichment pipeline | Low | Medium | Auto-approve low-risk changes (confidence > 0.95); escalation SLA for stale reviews (> 48 hrs) |
| PII exposure through vector embeddings | Low | High | Hash/tokenize PII before embedding (NFR-23); access controls on Vector DB; periodic security audit |
| Stakeholder misalignment on survivorship rules (which source wins) | Medium | High | Document field-level rules in Phase 3 spec; sign-off from data governance committee before implementation |
| SAP multi-role records misclassified as duplicates | Medium | High | Dedup engine must include SAP account_group awareness (FR-311); same legal entity with different roles resolved to single panw_customer_id; steward review for ambiguous cases |
| SAP integration increases ingest volume and schema complexity | Medium | Medium | SAP Account Type Mapper (FR-112) normalizes SAP-specific fields; account_roles table isolates role complexity from core staged_accounts schema |
| Sellable entity flag missing or incorrect in DNB/Zoom data | Medium | High | Benefit-of-doubt rule: if no DNB/Zoom entry exists, classify as customer (FR-113). Reclassification path (FR-212) handles corrections when late data arrives. Steward approval required for all reclassifications. |
| Location records incorrectly promoted to golden records | Low | High | Sellable entity gate (FR-313) enforces hard block at Phase 3. Only `record_class: customer` or SAP-typed records pass. Locations are linked via `location_of` edge, never merged into golden. |

---

## 11. Assumptions & Dependencies

### Assumptions
1. Salesforce remains the primary CRM and source of manual account creation.
2. DNB and Zoom data feeds are available via API or batch file delivery at least daily.
3. Google Cloud Platform (Vector DB, BigQuery) is the approved cloud platform.
4. Gemini Pro API is available with sufficient quota for enrichment workload.
5. The PANW Customer ID namespace does not collide with any existing internal identifier schemes.
6. SAP ERP integration is a future-state enhancement; current phases are designed to accommodate SAP without architectural rework.
7. A single legal entity in SAP may appear as multiple partner records with different account types (ZEND, ZSLD, ZSHP, ZBIL, ZPYR); these must resolve to one golden record.

### Dependencies
1. Access to Zoom data feed (API credentials, data-sharing agreement).
2. Google Vector DB provisioning and capacity allocation.
3. Gemini Pro API access and quota approval.
4. Salesforce API access with sufficient daily API call limits for auto-sync.
5. Data governance committee sign-off on survivorship rules (Phase 3).
6. Legal confirmation on data residency requirements (NFR-22).
7. *(Future)* SAP ERP access via IDoc / RFC / OData interface with appropriate credentials and data-sharing agreement.
8. *(Future)* SAP account type taxonomy mapping agreed with SAP admin team (ZEND, ZSLD, ZSHP, ZBIL, ZPYR, ZPRS → MDM normalized types).

---

## 12. Document Inventory

### Core Documents

| Document | Description |
|----------|-------------|
| [req.md](req.md) | This file — requirements, NFRs, risks, glossary |
| [data_model.md](data_model.md) | Cross-phase schema DDL, field evolution, ER diagram |
| [consolidation_report.md](consolidation_report.md) | Status summary and decision log |

### Phase Documents (main flow diagrams + prose)

| Document | Description |
|----------|-------------|
| [phase1_data_ingest_flow.md](phase1_data_ingest_flow.md) | Ingest pipeline, schema normalization, dedup pre-check, dual persistence |
| [phase2_data_enrichment_flow.md](phase2_data_enrichment_flow.md) | AI enrichment, hierarchy inference, governance, Salesforce/SAP sync |
| [phase3_data_master_flow.md](phase3_data_master_flow.md) | Deduplication, survivorship, golden record, downstream distribution |

### Supporting Diagram Files

| Document | Phase | Diagram Type |
|----------|-------|-------------|
| [phase1_error_handling.md](phase1_error_handling.md) | 1 | Error flow + DLQ management |
| [phase1_migration_cutover.md](phase1_migration_cutover.md) | 1 | Gantt timeline for migration |
| [phase1_cdc_sync.md](phase1_cdc_sync.md) | 1 | CDC sync architecture (BQ ↔ Vector DB) |
| [phase2_gemini_fallback.md](phase2_gemini_fallback.md) | 2 | Circuit breaker + fallback strategy |
| [phase2_late_arriving.md](phase2_late_arriving.md) | 2 | Late-arriving data sequence diagram |
| [phase2_governance_ui.md](phase2_governance_ui.md) | 2 | Governance state machine |
| [phase2_sap_impact.md](phase2_sap_impact.md) | 2 | SAP before/after on enriched_accounts |
| [phase3_sap_impact.md](phase3_sap_impact.md) | 3 | SAP before/after on golden_accounts |
| [phase3_sap_multi_role.md](phase3_sap_multi_role.md) | 3 | SAP multi-role entity resolution |
| [phase3_hierarchy_graph.md](phase3_hierarchy_graph.md) | 3 | Sample hierarchy tree visualization |
| [phase3_survivorship_tree.md](phase3_survivorship_tree.md) | 3 | Survivorship decision tree |
| [phase3_event_bus.md](phase3_event_bus.md) | 3 | Kafka event bus + consumer architecture |
| [phase3_cyberarch_flow.md](phase3_cyberarch_flow.md) | 3 | CyberArch match flow |

---

## 13. Glossary

| Term | Definition |
|------|-----------|
| **FSAII** | Few-Shot Adaptive Inference Indexing — AI search technique generating contextual vector embeddings for semantic similarity matching. |
| **PANW Customer ID** | Internal persistent identifier assigned at ingest (Phase 1) to link a record through enrichment and mastering. |
| **PANW Golden ID** | Master identifier assigned in Phase 3 representing a single deduplicated customer entity. |
| **D-U-N-S** | Data Universal Numbering System — a 9-digit identifier assigned by Dun & Bradstreet to uniquely identify businesses. |
| **Survivorship** | The process of selecting the best attribute values from multiple duplicate records to form a single master record. |
| **SCD Type 2** | Slowly Changing Dimension Type 2 — a versioning technique that preserves historical records by creating new rows for changes. |
| **Golden Record** | The single, authoritative master record for a customer entity after deduplication and survivorship. |
| **Governance UI** | Web interface for data stewards to review, approve, or reject enrichment changes before they sync to downstream systems. |
| **CyberArch** | Internal matching/search consumer system that queries customer data for identity resolution. |
| **cssot_** | Customer Single Source of Truth — field prefix convention for schema-harmonized attributes. |
| **SAP Account Group** | SAP classification code for customer account types: ZEND (End Customer), ZSLD (Sold-To), ZSHP (Ship-To), ZBIL (Bill-To), ZPYR (Payer), ZPRS (Prospect). |
| **Partner Function** | SAP concept where a single legal entity can hold multiple business roles (sold-to, ship-to, bill-to, payer) in the same transaction. |
| **IDoc** | Intermediate Document — SAP standard format for asynchronous data exchange between SAP and external systems. |
| **account_roles** | MDM table mapping partner functions to entities, preserving SAP role semantics while linking to a single `panw_customer_id`. |
| **golden_partner_roles** | Phase 3 table mapping SAP partner roles to golden record IDs, enabling role-based graph traversal. |
| **Sellable Entity** | A DNB/Zoom attribute indicating that a company has autonomous decision-making ability and is a valid customer target. Used to distinguish customers from locations during ingest classification. |
| **record_class** | MDM classification assigned at ingest: `customer` (eligible for golden record) or `location` (not eligible, linked to parent customer). Determined by DNB/Zoom sellable entity flag. |
| **Location Record** | A record classified as non-sellable by DNB/Zoom (for Salesforce-sourced data). Stored in the MDM but never promoted to a golden record. Linked to a parent customer via `location_of` edge in the Golden ID Graph. |
