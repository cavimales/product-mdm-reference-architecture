# ADR-0003: One canonical product schema at the STEP boundary

- **Status:** Accepted
- **Date:** 2026-10-18

## Context

Each source has its own item format, keys and units. If STEP accepted each format, its configuration would grow with every new source.

## Options considered

1. STEP imports each source format directly
2. One canonical, versioned schema; integration layer translates

## Decision

One canonical, versioned product schema in a schema registry. Source-specific formats stop at the integration layer.

## Consequences

- Upfront modelling effort
- Adding a source means a new adapter, not STEP configuration
- Schema changes follow a compatibility policy
