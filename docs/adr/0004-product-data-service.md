# ADR-0004: Separate read model for high-volume lookups

- **Status:** Accepted
- **Date:** 2026-10-18

## Context

E-commerce and order processing need fast, frequent product lookups. Serving them from STEP would load the MDM platform with read traffic.

## Options considered

1. Consumers call STEP APIs directly
2. A product data service with cache, fed from golden events

## Decision

A product data service, exposed through the API gateway and kept current from the golden topic.

## Consequences

- Eventual consistency of a few seconds
- STEP performance is protected
- One more service to run
