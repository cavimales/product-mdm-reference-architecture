# ADR-0001: MDM implementation style

- **Status:** Accepted
- **Date:** 2026-10-18

## Context

FreshLine creates product data in two legacy ERPs, and suppliers send new items by spreadsheet and email. New items need governed onboarding, while existing ERP items must keep flowing during migration.

## Options considered

1. Registry: STEP only indexes source records; sources stay authoritative
2. Consolidation: sources author, STEP builds golden records for downstream use
3. Coexistence: authoring in sources and STEP, golden values synced back
4. Centralized: all authoring in STEP, sources only consume

## Decision

Centralized for new items, created by suppliers in the STEP supplier portal. Consolidation for legacy ERP items until each source is cut over.

## Consequences

- New items are governed from the first submission
- Two operating modes run side by side for a period
- Each source needs an explicit cutover date and reconciliation
