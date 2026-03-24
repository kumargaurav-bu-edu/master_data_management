## Main 1
flowchart LR
    %% ==================== SOURCES ====================
    subgraph SRC["Source Systems"]
        SF["Salesforce"]
        DNB["D&B"]
        ZM["Zoom"]
        SAP["SAP (Future)"]
    end

    %% ==================== INGEST ====================
    subgraph INGEST["Ingest & Normalize"]
        IC["Ingest Controller"]
        SN["Schema Normalizer"]
    end

    %% ==================== MERGE ====================
    subgraph MERGE["Cross-Source Merge"]
        ME["Merge Engine"]
    end

    %% ==================== CLASSIFICATION ====================
    subgraph CLASSIFY["Sellable Entity Classification"]
        SE{"Sellable Entity?"}
        CUST["Customer"]
        LOC["Location"]
        SAPC["SAP Classification"]
    end

    %% ==================== DEDUP ====================
    subgraph DEDUP["Dedup Pre-Check"]
        MATCH["Vector Match Engine"]
        REVIEW["Review Queue"]
    end

    %% ==================== STORAGE ====================
    subgraph STORE["Persistence"]
        VDB["Vector DB"]
        BQ["BigQuery"]
    end

    %% ==================== FLOW ====================
    SF & DNB & ZM & SAP --> IC
    IC --> SN
    SN --> ME
    ME --> SE
    SE -->|"Customer"| MATCH
    SE -->|"Location"| BQ
    SE -->|"SAP"| MATCH
    MATCH -->|"No Match"| BQ
    MATCH -->|"Duplicate"| REVIEW
    REVIEW --> BQ
    BQ --> VDB

    %% ==================== SAMPLE RECORD CALLOUTS ====================

    R1["Sample Input
    name: Acme
    duns: null"] -.-> SN

    R2["After Merge
    name: Acme International
    duns: 999999999"] -.-> ME

    R3["Classification
    record_class: CUSTOMER"] -.-> SE

    R4["Dedup Check
    similarity: 0.91"] -.-> MATCH

    R5["Final Stored
    panw_id: PANW-0000-123"] -.-> BQ

    %% ==================== STYLING ====================
    style SRC fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style INGEST fill:#fff3e0,stroke:#fb8c00,stroke-width:2px
    style MERGE fill:#fce4ec,stroke:#e91e63,stroke-width:2px
    style CLASSIFY fill:#e1f5fe,stroke:#0288d1,stroke-width:2px
    style DEDUP fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px
    style STORE fill:#e0f2f1,stroke:#00897b,stroke-width:2px

    style SE fill:#fff9c4,stroke:#f9a825,stroke-width:3px

    %% Make sample records subtle
    style R1 fill:#ffffff,stroke:#9e9e9e,stroke-dasharray: 3 3
    style R2 fill:#ffffff,stroke:#9e9e9e,stroke-dasharray: 3 3
    style R3 fill:#ffffff,stroke:#9e9e9e,stroke-dasharray: 3 3
    style R4 fill:#ffffff,stroke:#9e9e9e,stroke-dasharray: 3 3
    style R5 fill:#ffffff,stroke:#9e9e9e,stroke-dasharray: 3 3


## Main 2

flowchart LR

  %% Sources
  subgraph SRC["Source Systems"]
    SF["Salesforce"]
    DNB["DNB"]
    ZM["Zoom"]
    SAP["SAP (Future)"]
  end

  %% Ingest
  subgraph INGEST["Ingest & Normalize"]
    IC["Ingest Controller"]
    SN["Schema Normalizer"]
  end

  %% Merge
  subgraph MERGE["Cross-Source Merge"]
    ME["Merge Engine"]
  end

  %% Classification
  subgraph CLASSIFY["Sellable Entity Classification"]
    SE{"Sellable Entity?"}
    CUST["Customer"]
    LOC["Location"]
    SAPC["SAP Classification"]
  end

  %% Dedup
  subgraph DEDUP["Dedup Pre-Check"]
    MATCH["Vector Match Engine"]
    REVIEW["Review Queue"]
  end

  %% Storage
  subgraph STORE["Persistence"]
    VDB["Vector DB"]
    BQ["BigQuery"]
  end

  %% Flow
  SF --> IC
  DNB --> IC
  ZM --> IC
  SAP --> IC

  IC --> SN
  SN --> ME
  ME --> SE

  SE -->|Customer| MATCH
  SE -->|Location| BQ
  SE -->|SAP| MATCH

  MATCH -->|No Match| BQ
  MATCH -->|Duplicate| REVIEW

  REVIEW --> BQ

  BQ --> VDB


## Data Sample 1
flowchart LR
    %% ================= BEFORE =================
    subgraph BEFORE["📥 Before Processing (Raw Input)"]
        A["source: Salesforce<br/>name: Acme Intl HQ<br/>website: acme.com<br/>zoom_id: null<br/>address: San Jose, CA"]
    end

    %% ================= AFTER MERGE =================
    subgraph AFTER["🔗 After Merge (Zoom Enrichment)"]
        B["name: Acme International<br/>website: acme.com<br/>zoom_id: ZOOM-801<br/>industry: Manufacturing<br/>source: SF + Zoom"]
    end

    %% ================= CLASSIFICATION =================
    subgraph CLASSIFY["🔍 Classification"]
        C{"Sellable Entity?"}
    end

    %% ================= CUSTOMER PATH =================
    subgraph CUSTOMER["✅ Customer (Sellable Entity)"]
        C1["record_class: CUSTOMER<br/>panw_id: PANW-0000-123<br/>zoom_id: ZOOM-801<br/>eligible_for_golden: YES"]
        V1["Vector DB<br/>embedding: [vector]<br/>metadata: panw_id, name, website, zoom_id"]
        BQ1["BigQuery: staged_customers<br/>panw_id: PANW-0000-123<br/>name: Acme International<br/>zoom_id: ZOOM-801<br/>status: staged"]
    end

    %% ================= LOCATION PATH =================
    subgraph LOCATION["📍 Location (Non-Sellable)"]
        L1["record_class: LOCATION<br/>panw_id: PANW-0000-456<br/>parent_customer: PANW-0000-123"]
        V2["Vector DB<br/>embedding: [vector]<br/>metadata: panw_id, name, address, type: location"]
        BQ2["BigQuery: staged_locations<br/>panw_id: PANW-0000-456<br/>name: Acme Intl HQ<br/>address: San Jose, CA<br/>parent_customer_id: PANW-0000-123<br/>status: staged"]
    end

    %% ================= FLOW CONNECTIONS =================
    A --> B
    B --> C
    C -->|"Customer"| C1
    C -->|"Location"| L1
    C1 --> V1
    C1 --> BQ1
    L1 --> V2
    L1 --> BQ2

    %% ================= STYLING =================
    style BEFORE fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style AFTER fill:#fff3e0,stroke:#fb8c00,stroke-width:2px
    style CLASSIFY fill:#e1f5fe,stroke:#0288d1,stroke-width:2px
    style CUSTOMER fill:#e8f5e9,stroke:#43a047,stroke-width:2px
    style LOCATION fill:#ffebee,stroke:#f44336,stroke-width:2px

    style C fill:#fff9c4,stroke:#f9a825,stroke-width:3px
    style C1 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style L1 fill:#ffcdd2,stroke:#e53935,stroke-width:2px


## Data Sample 2

flowchart LR

    %% ================= BEFORE =================
    subgraph BEFORE["Before Processing (Raw Input)"]
        A["source: Salesforce
name: Acme Intl HQ
website: acme.com
zoom_id: null
address: San Jose, CA"]
    end

    %% ================= AFTER MERGE =================
    subgraph AFTER["After Merge (Zoom Enrichment)"]
        B["name: Acme International
website: acme.com
zoom_id: ZOOM-801
industry: Manufacturing
source: SF + Zoom"]
    end

    %% ================= CLASSIFICATION =================
    subgraph CLASSIFY["Classification"]
        C{"Sellable Entity?"}
    end

    %% ================= CUSTOMER PATH =================
    subgraph CUSTOMER["Customer (Sellable Entity)"]
        C1["record_class: CUSTOMER
panw_id: PANW-0000-123
zoom_id: ZOOM-801
eligible_for_golden: YES"]

        V1["Vector DB
embedding: [vector]
metadata:
  panw_id: PANW-0000-123
  name: Acme International
  website: acme.com
  zoom_id: ZOOM-801"]

        BQ1["BigQuery: staged_customers
panw_id: PANW-0000-123
name: Acme International
zoom_id: ZOOM-801
industry: Manufacturing
status: staged"]
    end

    %% ================= LOCATION PATH =================
    subgraph LOCATION["Location (Non-Sellable)"]
        L1["record_class: LOCATION
panw_id: PANW-0000-456
parent_customer: PANW-0000-123"]

        V2["Vector DB
embedding: [vector]
metadata:
  panw_id: PANW-0000-456
  name: Acme Intl HQ
  address: San Jose, CA
  type: location"]

        BQ2["BigQuery: staged_locations
panw_id: PANW-0000-456
name: Acme Intl HQ
address: San Jose, CA
parent_customer_id: PANW-0000-123
status: staged"]
    end

    %% ================= FLOW =================
    A --> B --> C
    C -->|Customer| C1
    C -->|Location| L1

    C1 --> V1
    C1 --> BQ1

    L1 --> V2
    L1 --> BQ2