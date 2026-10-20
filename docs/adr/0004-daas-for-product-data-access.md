# ADR-0004: Stibo DaaS for on-demand product data access

- **Status:** Accepted
- **Date:** 2026-10-20

## Context

E-commerce, order entry and other applications need fast, frequent product lookups. Serving them from STEP's core would load the MDM platform with read traffic and tie their availability to STEP maintenance windows.

## Options considered

1. Consumers call STEP's core APIs directly
2. A custom product data service with its own store and cache, fed from golden events
3. Stibo STEP Data as a Service (DaaS): STEP publishes into a separate read store, exposed through GraphQL APIs

## Decision

Use DaaS for on-demand reads, exposed to consumers through the API gateway. Consumers that need data in bulk subscribe to the golden Kafka topic instead.

## Consequences

- No custom read service to build, host or keep in sync
- STEP's core is protected from high-volume read traffic
- Consumers must use GraphQL, and data is eventually consistent with STEP
- The read path depends on the vendor; DaaS capabilities and limits must be validated against key consumer queries early
- A custom read service (option 2) remains the fallback if DaaS cannot meet a consumer's needs
