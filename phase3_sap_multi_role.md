# Phase 3: SAP Multi-Role Entity Resolution

> Extracted from [phase3_data_master_flow.md](phase3_data_master_flow.md)

The key challenge with SAP is that a single legal entity (e.g., Acme International) can appear as multiple SAP partner records with different roles. The MDM must recognize these are the **same entity** with **different roles**, not duplicates.

```mermaid
flowchart TB
  subgraph SAP_RECORDS["SAP Source Records - Same Legal Entity"]
    direction LR
    S1["SAP-90001\nZEND: End Customer\nAcme International\nCompany Code: 1000"]
    S2["SAP-90001\nZSLD: Sold-To Party\nAcme International\n Sales Org: US01"]
    S3["SAP-90002\nZSHP: Ship-To Party\nAcme International\nPlant: US-WEST"]
    S4["SAP-90003\nZBIL: Bill-To Party\nAcme International\nPayment Terms: NET30"]
    S5["SAP-90004\nZPYR: Payer\nAcme International\nCredit Limit: 500K"]
  end

  subgraph RESOLVE["MDM Entity Resolution"]
    DETECT["fa:fa-search Entity Detection\n─────────────\nAll 5 SAP records matched to\nsame legal entity via:\n- name similarity: 1.0\n- SAP account_group linkage\n- company_code correlation"]
    SINGLE["fa:fa-id-card Single PANW Customer ID\n─────────────\npanw_customer_id: PANW-0000-123\nAll 5 SAP roles linked\nto same staging record"]
  end

  subgraph GOLDEN["Golden Record"]
    GOLD["fa:fa-trophy PANW-GOLD-000001\nAcme International\n─────────────\naccount_type: end_customer\nmaster_status: golden"]
    ROLES["Partner Role Registry\n─────────────\nsold_to: self\nship_to: PANW-GOLD-000066\n  (Acme West Warehouse)\nbill_to: self\npayer: self"]
  end

  S1 --> DETECT
  S2 --> DETECT
  S3 --> DETECT
  S4 --> DETECT
  S5 --> DETECT
  DETECT --> SINGLE
  SINGLE --> GOLD
  GOLD --> ROLES

  style SAP_RECORDS fill:#fff8e1,stroke:#FF6F00,stroke-width:2px,stroke-dasharray: 8 4
  style RESOLVE fill:#f3e5f5,stroke:#9C27B0,stroke-width:2px
  style GOLDEN fill:#e8f5e9,stroke:#4CAF50,stroke-width:2px
```
