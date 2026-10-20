# Context diagram

STEP masters the data and hosts supplier onboarding; Kafka connects the ERPs and bulk consumers; DaaS serves on-demand reads.

```mermaid
flowchart LR
    subgraph Sources
        ERPA["ERP-A (legacy)<br/>Change capture"]
        ERPB["ERP-B (acquired)<br/>Change capture"]
        SUPP["Suppliers"]
    end

    subgraph Integration["Integration layer"]
        KAFKA["Kafka backbone<br/>Canonical product schema<br/>Schema registry<br/>Dead-letter topics, replay<br/>ERP write-back adapters"]
    end

    subgraph Stibo["Stibo STEP platform"]
        PORTAL["Supplier portal<br/>Supplier-scoped access"]
        STEP["STEP core<br/>Matching, golden record, DQ<br/>Stewardship workflows"]
        DAAS["Data as a Service<br/>GraphQL read APIs"]
    end

    subgraph Consumers
        GW["API gateway<br/>Auth and throttling"]
        APPS["Applications<br/>E-commerce, order entry"]
        DWH["Analytical warehouse<br/>History, KPIs, DQ trends"]
        OTHER["Other subscribers<br/>Warehouse management"]
    end

    ERPA <--> KAFKA
    ERPB <--> KAFKA
    SUPP --> PORTAL
    PORTAL --> STEP
    KAFKA <--> STEP
    STEP --> DAAS
    APPS --> GW --> DAAS
    KAFKA --> DWH
    KAFKA --> OTHER

    classDef mdm fill:#e6f0ff,stroke:#2f6fdb,stroke-width:2px;
    class STEP mdm;
```

ERPs publish changes in and receive golden data back through adapters on the backbone. Suppliers work in STEP's supplier portal. Applications query DaaS through the gateway, and bulk consumers such as the analytical warehouse subscribe to golden events.
