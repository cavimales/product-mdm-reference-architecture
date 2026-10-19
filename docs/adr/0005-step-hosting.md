# ADR-0005: Vendor-hosted STEP SaaS

- **Status:** Accepted
- **Date:** 2026-10-18

## Context

FreshLine must choose between running STEP on its own infrastructure or using the vendor-hosted service.

## Options considered

1. Self-hosted STEP on FreshLine's cloud
2. Vendor-hosted STEP SaaS

## Decision

Vendor-hosted SaaS.

## Consequences

- Less operational burden for the platform team
- Less control over upgrade timing and infrastructure
- Network connectivity and authentication to Kafka must be agreed with the vendor
