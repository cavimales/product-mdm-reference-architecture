# ADR-0007: STEP supplier portal for supplier onboarding

- **Status:** Accepted
- **Date:** 2026-10-20

## Context

New items and their identity and logistics attributes originate with suppliers. Today these arrive as spreadsheets and email, which slows setup and introduces errors.

## Options considered

1. A custom supplier portal integrated with STEP through an API gateway and Kafka
2. A third-party supplier collaboration product integrated with STEP
3. Stibo STEP's supplier portal (Supplier Web UI)

## Decision

Use the STEP supplier portal. Suppliers sign in with supplier-scoped accounts and enter items directly into STEP onboarding workflows.

## Consequences

- No portal, API or integration to build for supplier submissions
- Supplier edits enter governed workflows directly, with business rule feedback in the portal
- Less control over the supplier user experience than a custom portal
- Supplier account administration and supplier training become an operational responsibility
- Large suppliers that want system-to-system exchange still need a file or API-based path, to be designed when required
