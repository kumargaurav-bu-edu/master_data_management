# MDM Project — Chat Context Summary

> Use this file to onboard a new Cursor chat in any project. Point the new chat to:
> `Read /Users/gaurav/workspace/modelling/mdm/CONTEXT.md`
> Then say: "Read all the files listed in the inventory below and continue helping me with this MDM solution."

---

## What This Project Is

A 3-phase Master Data Management (MDM) solution for customer account data:
- **Phase 1**: Data Ingest — migrate from BigQuery+Elastic to Vector DB+FSAII AI search, onboard Zoom+SAP, assign PANW Customer ID
- **Phase 2**: Data Enrichment — Gemini Pro AI enrichment, hierarchy inference, governance UI with steward approval, Salesforce/SAP auto-sync
- **Phase 3**: Data Mastering — deduplication, field-level survivorship, golden record, relationship graph, Kafka distribution

## Key Architecture Decisions

1. **Identity chain**: `source_id` → `panw_customer_id` (Phase 1) → `panw_golden_id` (Phase 3)
2. **Storage**: Google Vector DB (AI search) + BigQuery (structured analytics), CDC-synced via Pub/Sub
3. **AI**: FSAII for vector search, Gemini Pro for enrichment with circuit-breaker fallback to rule-based
4. **Sellable Entity**: DNB/Zoom provide `sellable_entity` flag. Salesforce records classified as `customer` (sellable/no entry) or `location` (not sellable). SAP retains its own types. Only customers become golden records.
5. **SAP Multi-Role**: SAP partner functions (End Customer, Sold-To, Ship-To, Bill-To, Payer) resolve to one `panw_customer_id` — roles, not duplicates
6. **Survivorship**: Field-level rules — trust (SF>SAP>DNB>Zoom), recency (phone/email), confidence (industry/segment)
7. **SCD Type 2**: Applied in Phase 2 (enriched_accounts) and Phase 3 (golden_accounts) for versioning and rollback

## File Inventory

All files are at: `/Users/gaurav/workspace/modelling/mdm/`

### Core Documents
| File | Lines | Description |
|------|-------|-------------|
| `req.md` | ~340 | Requirements (FRs, NFRs, risks, metrics, glossary, document inventory) |
| `data_model.md` | ~313 | Schema DDL for all tables, field evolution, ER diagram, identity chain |
| `consolidation_report.md` | ~105 | Status summary and decision log |

### Phase Documents (each has one main Mermaid diagram + prose)
| File | Lines | Description |
|------|-------|-------------|
| `phase1_data_ingest_flow.md` | ~334 | Ingest pipeline with SAP dotted integration, sellable entity classification, DB view showing 1 SF + 5 SAP records |
| `phase2_data_enrichment_flow.md` | ~334 | Enrichment pipeline with SAP, sellable entity reclassification on late-arriving data |
| `phase3_data_master_flow.md` | ~410 | Golden record pipeline with SAP, sellable entity gate, location records |

### Supporting Diagram Files (extracted to keep phase files small)
| File | Phase | Content |
|------|-------|---------|
| `phase1_error_handling.md` | 1 | Error flow + DLQ |
| `phase1_migration_cutover.md` | 1 | Gantt timeline |
| `phase1_cdc_sync.md` | 1 | CDC architecture |
| `phase2_gemini_fallback.md` | 2 | Circuit breaker + fallback |
| `phase2_late_arriving.md` | 2 | Sequence diagram |
| `phase2_governance_ui.md` | 2 | State machine |
| `phase2_sap_impact.md` | 2 | Before/after enriched_accounts |
| `phase3_sap_impact.md` | 3 | Before/after golden_accounts |
| `phase3_sap_multi_role.md` | 3 | SAP entity resolution |
| `phase3_hierarchy_graph.md` | 3 | Hierarchy tree |
| `phase3_survivorship_tree.md` | 3 | Decision tree |
| `phase3_event_bus.md` | 3 | Kafka architecture |
| `phase3_cyberarch_flow.md` | 3 | Match flow |

## Key Concepts to Know

- **FSAII**: Few-Shot Adaptive Inference Indexing — AI vector search replacing Elasticsearch
- **cssot_**: Customer Single Source of Truth — field prefix for harmonized schema
- **Sellable Entity**: DNB/Zoom flag. If true or no entry → customer. If false + SF source → location. SAP exempt.
- **record_class**: `customer` (golden-eligible) or `location` (not golden-eligible, linked to parent)
- **SAP Account Groups**: ZEND (End Customer), ZSLD (Sold-To), ZSHP (Ship-To), ZBIL (Bill-To), ZPYR (Payer), ZPRS (Prospect)
- **CyberArch**: Internal match/search consumer with dedicated API (HIGH/POSSIBLE/LOW confidence tiers)
- **Governance UI**: Steward review + approve/reject with auto-approve for high-confidence changes

## What Has Been Completed

- [x] Requirements restructured with 40+ FRs, 23 NFRs, traceability to challenges
- [x] Data model with 10 table schemas, field evolution, ER diagram
- [x] 3 main Mermaid diagrams (one per phase) with SAP dotted integration + sellable entity logic
- [x] 13 supporting diagrams extracted to separate files
- [x] Error handling, failure modes, DLQ design for all phases
- [x] Integration architecture (connectors, protocols, CDC)
- [x] Migration/cutover plan with shadow mode and rollback
- [x] Field-level survivorship rules with decision tree
- [x] CyberArch match API spec with request/response examples
- [x] SAP multi-role entity resolution across all phases
- [x] Sellable entity classification, reclassification, and golden record gate
- [x] Success metrics with baselines and per-phase targets
- [x] Risk register with mitigations

## Suggested Prompt for New Chat

```
Read /Users/gaurav/workspace/modelling/mdm/CONTEXT.md to understand my MDM project,
then read req.md and the three phase files. I want to continue working on this.
[your specific request here]
```
