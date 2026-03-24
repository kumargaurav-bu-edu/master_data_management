# Main 
flowchart TB
    %% ================= INPUT =================
    subgraph INPUT["📥 Staged Input (Phase 1)"]
        A["Staged Record<br/>panw_id: PANW-0000-123<br/>name: Acme International<br/>website: acme.com<br/>zoom_id: ZOOM-801"]
        SAP_IN["SAP Record<br/>sap_account_id: SAP-90001<br/>account_type: end_customer<br/>linked: PANW-0000-123"]
    end

    %% ================= REFERENCES =================
    subgraph REF["🔎 Reference Sources"]
        DNB["DNB<br/>duns: 999999999<br/>industry: Manufacturing"]
        ZM["Zoom<br/>zoom_id: ZOOM-801<br/>firmographics: Enterprise"]
        SFHINT["Salesforce Hint<br/>region: US<br/>parent: Acme Parent"]
        SAP_REF["SAP Master Data"]
    end

    %% ================= ENRICH =================
    subgraph ENRICH["🔗 Enrichment"]
        MERGE["Merge Engine<br/>combine all sources<br/>resolve entity"]
        AI["AI Enrichment<br/>+ industry<br/>+ company_type<br/>+ hierarchy<br/>confidence score"]
    end

    %% ================= VALIDATE =================
    subgraph VALIDATE["🔍 Quality Gate"]
        QR{"Valid?"}
    end

    %% ================= RECLASS =================
    subgraph RECLASS["🔄 Reclassification"]
        SE{"Sellable Entity Change?"}
        DOWN["Customer → Location"]
        UP["Location → Customer"]
        KEEP["No Change"]
        SAP_KEEP["SAP Type Retained"]
    end

    %% ================= GOVERN =================
    subgraph GOVERN["👥 Governance"]
        GOV["Steward Review"]
        APPROVED["Approved Record<br/>industry: Manufacturing<br/>confidence: 0.95"]
        REJECT["Rejected"]
    end

    %% ================= STORAGE =================
    subgraph STORE["💾 Storage + Sync"]
        VDB["Vector DB<br/>embedding + metadata"]
        BQ["BigQuery<br/>enriched_accounts<br/>SCD2"]
        SFAPI["Salesforce Sync"]
        SAP_SYNC["SAP Sync"]
    end

    METRICS["📊 Metrics<br/>success rate<br/>latency<br/>duplicate rate"]

    %% ================= FLOW =================
    A & SAP_IN --> MERGE
    DNB & ZM & SFHINT & SAP_REF --> MERGE
    MERGE --> AI
    AI --> QR
    QR -->|"Pass"| SE
    QR -->|"Review"| GOV
    SE --> DOWN
    SE --> UP
    SE --> KEEP
    SE --> SAP_KEEP
    DOWN & UP --> GOV
    KEEP & SAP_KEEP --> APPROVED
    GOV -->|"Approve"| APPROVED
    GOV -->|"Reject"| REJECT
    APPROVED --> VDB & BQ & SFAPI & SAP_SYNC & METRICS

    %% ================= STYLING =================
    style INPUT fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style REF fill:#fff8e1,stroke:#f57c00,stroke-width:2px
    style ENRICH fill:#fce4ec,stroke:#e91e63,stroke-width:2px
    style VALIDATE fill:#e1f5fe,stroke:#0288d1,stroke-width:2px
    style RECLASS fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px
    style GOVERN fill:#fff9c4,stroke:#f9a825,stroke-width:2px
    style STORE fill:#e0f2f1,stroke:#00897b,stroke-width:2px
    style METRICS fill:#f5f5f5,stroke:#455a64,stroke-width:2px

    style QR fill:#fff9c4,stroke:#f9a825,stroke-width:3px
    style SE fill:#fff9c4,stroke:#f9a825,stroke-width:3px
    style APPROVED fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px

# Compact
flowchart LR
    INPUT["📥 Input<br/>Staged + SAP Record"] 
    --> MERGE["🔗 Merge Engine"]

    REF["🔎 DNB + Zoom + SF + SAP"] 
    --> MERGE

    MERGE --> AI["🤖 AI Enrichment"]
    AI --> QR{"Valid?"}

    QR -->|"Pass"| SE{"Sellable Change?"}
    QR -->|"Review"| GOV["👥 Steward Review"]

    SE --> DOWN["Cust→Loc"] & UP["Loc→Cust"] & KEEP["Keep"]
    DOWN & UP --> GOV
    KEEP --> APPROVED["✅ Approved"]

    GOV -->|"Approve"| APPROVED
    GOV -->|"Reject"| REJECT["❌ Rejected"]

    APPROVED --> STORE["💾 Vector DB + BigQuery + SF Sync + SAP Sync"]
    APPROVED --> METRICS["📊 Metrics"]

    %% Styling
    style INPUT fill:#e3f2fd,stroke:#1976d2
    style REF fill:#fff8e1,stroke:#f57c00
    style MERGE fill:#fce4ec,stroke:#e91e63
    style AI fill:#e1f5fe,stroke:#0288d1
    style QR fill:#fff9c4,stroke:#f9a825,stroke-width:3px
    style SE fill:#fff9c4,stroke:#f9a825,stroke-width:3px
    style GOV fill:#fff9c4,stroke:#f9a825
    style APPROVED fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
    style STORE fill:#e0f2f1,stroke:#00897b
    style METRICS fill:#f5f5f5,stroke:#455a64