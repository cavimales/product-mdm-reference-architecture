# ADR-0002: Kafka as the integration backbone

- **Status:** Accepted
- **Date:** 2026-10-18

## Context

STEP must exchange product data with two ERPs and several consumers. Point-to-point links would multiply formats, retries and monitoring.

## Options considered

1. Point-to-point integrations from STEP to each system
2. An ESB or iPaaS orchestrating every flow
3. Kafka between STEP and the ERPs and bulk consumers, using STEP's native Kafka inbound and outbound endpoints

## Decision

Kafka between STEP, the ERPs and bulk consumers. No ERP or bulk consumer integrates with STEP directly.

## Consequences

- Replay, decoupling and one place to monitor
- More moving parts than direct integration
- Requires schema governance and topic ownership
