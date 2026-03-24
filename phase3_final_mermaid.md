flowchart TB
    %% ================= INPUT =================
    subgraph INPUT["📥 Enriched Records from Phase 2"]
        A["fa:fa-file-alt Enriched Record<br/>panw_customer_id: PANW-0000-123<br/>name: Acme International<br/>duns: 999999999<br/>industry: Manufacturing"]
        SAP_ENR["fa:fa-industry SAP Enriched<br/>sap_account_id: SAP-90001<br/>account_type: end_customer<br/>partner_functions: sold_to, ship_to..."]
    end

    %% ================= DEDUP & MATCH =================
    subgraph DEDUP["Step 1: Match + Score"]
        MATCH["fa:fa-search Dedup & Survivor Engine<br/>Vector + Deterministic Rules<br/>NEW: SAP multi-role awareness"]
    end

    %% ================= SELLABLE GATE =================
    subgraph GATE["Step 1b: Sellable Entity Gate"]
        SE{"record_class = customer<br/>or SAP-typed?"}
        LOC["Store as Location<br/>Linked to parent golden"]
    end

    %% ================= DECISIONS =================
    subgraph DECISIONS["Step 2: Merge Decisions"]
        NEW["fa:fa-plus Create New Golden<br/>Assign PANW-GOLD-000001"]
        MERGE["fa:fa-compress Physical Merge<br/>Survivorship: SF > SAP > DNB > Zoom"]
        REVIEW["fa:fa-user-check Steward Review<br/>Under Review"]
    end

    %% ================= MASTER STORE =================
    subgraph MASTER["Step 3: Golden Record Store"]
        SCD["fa:fa-database Master Record (SCD2)<br/>panw_golden_id: PANW-GOLD-000001<br/>status: golden<br/>NEW: account_type + partner_roles"]
    end

    %% ================= GRAPH & DISTRIBUTION =================
    subgraph GRAPH["Step 4: Relationship Graph"]
        GOLDEN_GRAPH["fa:fa-project-diagram Golden ID Graph<br/>NEW: SAP partner role edges"]
    end

    subgraph DISTRIBUTE["Step 5: Distribution"]
        KAFKA["fa:fa-stream Kafka Event Bus<br/>Topic: customer.golden.record"]
    end

    subgraph CONSUMERS["Step 6: Consumers"]
        API["fa:fa-plug Read APIs"]
        SF_SYNC["fa:fa-cloud Salesforce Sync"]
        SAP_SYNC["fa:fa-industry SAP Sync"]
        BI["fa:fa-chart-line BI + Analytics"]
    end

    %% ================= CONNECTIONS =================
    A & SAP_ENR --> MATCH
    MATCH -->|"≥ 0.95 Duplicate"| MERGE
    MATCH -->|"0.80–0.95 Possible"| REVIEW
    MATCH -->|"< 0.80 New"| NEW

    MATCH --> SE
    SE -->|"Location"| LOC
    SE -->|"Customer / SAP"| NEW

    NEW & MERGE & REVIEW --> SCD
    SCD --> GOLDEN_GRAPH
    SCD --> KAFKA
    KAFKA --> API & SF_SYNC & SAP_SYNC & BI

    %% ================= STYLING =================
    style INPUT fill:#e8f4fd,stroke:#2196F3,stroke-width:2px
    style DEDUP fill:#f3e5f5,stroke:#9C27B0,stroke-width:2px
    style GATE fill:#e1f5fe,stroke:#0288D1,stroke-width:2px
    style DECISIONS fill:#fff3e0,stroke:#FF9800,stroke-width:2px
    style MASTER fill:#e8f5e9,stroke:#4CAF50,stroke-width:2px
    style GRAPH fill:#e0f2f1,stroke:#009688,stroke-width:2px
    style DISTRIBUTE fill:#fce4ec,stroke:#E91E63,stroke-width:2px
    style CONSUMERS fill:#f5f5f5,stroke:#607D8B,stroke-width:2px

    style SAP_ENR fill:#fff8e1,stroke:#FF6F00,stroke-width:2px,stroke-dasharray:8 4
    style SAP_SYNC fill:#fff8e1,stroke:#FF6F00,stroke-width:2px,stroke-dasharray:8 4

    style SE fill:#fff9c4,stroke:#f9a825,stroke-width:3px