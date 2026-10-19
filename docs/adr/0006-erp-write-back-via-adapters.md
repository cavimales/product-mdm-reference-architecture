# ADR-0006: Golden data written back to ERPs by adapters

- **Status:** Accepted
- **Date:** 2026-10-18

## Context

ERPs need enriched golden data back, but each has its own API, keys and error behaviour.

## Options considered

1. STEP outbound endpoints write to each ERP directly
2. Per-ERP adapters consume the golden topic and write to the ERP

## Decision

Per-ERP adapters on the integration layer.

## Consequences

- ERP-specific logic stays out of STEP
- Adapters must be built and maintained
- Write-back failures need reconciliation and alerting
