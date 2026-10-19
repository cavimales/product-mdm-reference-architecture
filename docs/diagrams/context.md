# Context diagram

STEP masters the data; Kafka connects every system to it. No source or consumer talks to STEP directly.

```mermaid
flowchart LR
    subgraph Sources
        ERPA["ERP-A (legacy)<br/>change capture"]
        ERPB["ERP-B (acquired)<br/>change capture"]
        PORTAL["Supplier portal<br/>new item submissions"]
        GW["API gateway<br/>auth and throttling"]
    end

    subgraph Integration["Integration layer"]
        KAFKA["Kafka backbone<br/>Canonical product schema<br/>Schema registry<br/>Dead-letter topics, replay<br/>ERP write-back adapters"]
    end

    subgraph MDM["MDM and consumers"]
        STEP["Stibo STEP (SaaS)<br/>matching, golden record, DQ<br/>stewardship workflows"]
        PDS["Product data service<br/>fast lookups via gateway"]
        BQ["BigQuery<br/>history and DQ trends"]
        OTHER["Other consumers<br/>e-commerce, warehouse"]
    end

    ERPA <--> KAFKA
    ERPB <--> KAFKA
    PORTAL --> GW
    GW --> KAFKA
    KAFKA <--> STEP
    KAFKA --> PDS
    KAFKA --> BQ
    KAFKA --> OTHER

    classDef mdm fill:#e6f0ff,stroke:#2f6fdb,stroke-width:2px;
    class STEP mdm;
```

Sources publish changes in and receive golden data back through adapters on the backbone. STEP masters and governs. Consumers subscribe to golden events or read through the product data service.
