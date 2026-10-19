# Product MDM on Stibo STEP — Reference Architecture

*Version 0.1 · October 2026*

## 1. Context and goals

FreshLine Foods, a fictional food distributor, adopts Stibo STEP as its product MDM platform so that every system works from one governed product record. This document is the production target architecture: it decides what each layer owns and how they connect, not how STEP is configured internally.

**Business problem.** Product data is created in three places that disagree: a legacy ERP, an ERP inherited through acquisition, and a supplier onboarding portal. Duplicates, conflicting attributes and slow new-item setup delay sales and cause ordering errors.

**Goals**

- One trusted product record per item, with clear ownership of each attribute
- New item setup governed end to end, from supplier submission to ERP availability
- Product changes available to every consuming system within minutes
- Measurable data quality, reported by source and by attribute group

**Scope:** the product domain, its integrations and the production platform around STEP. **Out of scope:** STEP data model and business rule design, customer and supplier domains, and commercial considerations such as licensing.

**Note on sources.** The design uses only public Stibo documentation and a fictional company. Nothing is taken from any employer's implementation.

## 2. Architecture at a glance

No system talks to STEP directly: every source and consumer connects through the Kafka backbone or the API gateway, so STEP can be upgraded, scaled or replaced without touching them.

See [the context diagram](diagrams/context.md).

Sources publish changes in and receive golden data back through adapters on the backbone; STEP masters and governs; consumers subscribe to golden events or read through the product data service.

## 3. Building blocks

STEP owns mastering and governance; everything around it is deliberately thin, standard and replaceable, so STEP is never coupled directly to a source or consumer.

| Layer | Responsibility | Technology (proposed) |
| --- | --- | --- |
| Source systems | Create and change local item data; consume golden data back | ERP-A, ERP-B, supplier portal |
| Integration layer | Transport, routing, format translation, retries, replay | Managed Kafka with schema registry; integration services on GKE or Cloud Run |
| API gateway | Secure, throttled synchronous access to product data | Apigee or an equivalent gateway |
| MDM platform | Data model, matching and merge, golden record, business rules, stewardship workflows, data quality | Stibo STEP (vendor SaaS preferred) |
| Product data service | Low-latency read model of golden records for high-volume consumers | Service + cache fed from golden events |
| Analytics platform | History, data quality trends, reporting | BigQuery, loaded from golden and DQ events |
| Platform services | Identity, secrets, monitoring, CI/CD | Corporate IdP (SSO), Secret Manager, Cloud Monitoring, GitHub Actions |

## 4. Integration patterns

Kafka is the backbone in both directions, and STEP connects to it natively: inbound integration endpoints can read from a topic with the [Kafka Receiver](https://service.stibosystems.com/documentation/step2024q3/content/dataexc/iiep/creatingiiep/iiepreceivermethods/kafkarec.html), and outbound endpoints can publish with the [Kafka Delivery method](https://service.stibosystems.com/documentation/latest/content/dataexc/oiep/creating_an_oiep/oiepdeliverymethods/kafkadelmth.html).

| Flow | Pattern | Notes |
| --- | --- | --- |
| ERP item changes into STEP | ERP change capture → canonical event on Kafka → STEP inbound endpoint | Integration layer maps each ERP format to one canonical product schema before STEP sees it |
| Supplier new-item submission | Portal → API gateway → Kafka → STEP inbound endpoint → onboarding workflow | Submission acknowledged immediately; status returned to the portal through events |
| Golden record distribution | STEP outbound endpoint → Kafka golden topic → each consumer subscribes | One publication, many consumers; consumers never call STEP for bulk data |
| ERP enrichment write-back | Golden topic → per-ERP adapter → ERP API or staging tables | Adapters own each ERP's format, keys and error handling |
| Synchronous lookup | Consumer → API gateway → product data service | Shields STEP from high-volume reads |
| Analytics | Golden and DQ topics → BigQuery | Full change history for reporting and audit |

**Design rules**

- STEP exchanges one canonical, versioned schema; source-specific formats stop at the integration layer.
- Every message carries a source key and an event ID, so consumers can deduplicate and replay safely.
- Failed messages go to dead-letter topics with the error attached and are replayable after a fix.
- Bulk loads (initial migration, large supplier catalogs) use file-based import, not the event path.

## 5. Governance and stewardship

Every attribute group has one owning team and one system of entry, so conflicts are resolved by rule rather than by whoever wrote last.

| Attribute group | System of entry | Owner |
| --- | --- | --- |
| Identity (GTIN, brand, supplier item number) | Supplier portal | Supplier onboarding team |
| Descriptions and marketing content | STEP | Product content team |
| Logistics (dimensions, pack, pallet) | Supplier portal, verified in STEP | Supply chain data team |
| Commercial (category, status, sourcing) | STEP | Category management |
| ERP-specific settings | Each ERP | ERP operations |

**Stewardship in STEP**

- New items follow a STEP workflow: supplier submission, validation, enrichment, approval, then publication to ERPs.
- Probable duplicates go to a steward review task instead of merging automatically.
- Business rules block publication of records that fail mandatory checks.

**Data quality**

- Rules are defined once, in STEP, and their results are published with each record.
- BigQuery holds DQ results over time, reported by source system and attribute group.
- A data governance forum reviews DQ trends and rule changes monthly.

## 6. Production readiness

The platform is run like any tier-1 system: separate environments, automated releases, least-privilege access and monitoring that follows a product change end to end. Targets below are proposals to agree with the business.

| Area | Approach |
| --- | --- |
| Environments | Dev, test, pre-production and production for STEP and the integration layer; pre-production mirrors production volumes |
| Release management | STEP configuration promoted between environments through a controlled export and import process; integration services deployed through CI/CD with automated contract tests |
| Security | SSO for all STEP users with role-based access; internal systems authenticate with OAuth client credentials or mTLS; the supplier portal calls the API with its own service account and passes the logged-in supplier's ID as a signed claim, so the gateway can throttle, authorize and audit per supplier; suppliers integrating system-to-system get their own OAuth clients; encryption in transit and at rest; secrets in a vault, never in configuration files |
| Availability | Target 99.9% for STEP and the integration layer during business hours; Kafka retention lets STEP catch up after an outage without data loss |
| Disaster recovery | Proposed targets: RPO 1 hour, RTO 4 hours; tested twice a year |
| Performance | Golden record change visible to consumers within 5 minutes at p95; lookups served from the product data service, not STEP |
| Observability | Integration logs and Kafka consumer lag in one monitoring view; alerts on dead-letter growth, endpoint failures and workflow backlogs |
| Data migration | Phased cutover by source system; initial loads by bulk import, then switch to events; reconciliation reports before each cutover |
| Support model | Integration and platform teams on call; data stewards own workflow queues; vendor support for STEP platform issues |

## 7. Key decisions and trade-offs

Six decisions shape this architecture; each would be recorded as an ADR with the options considered.

| # | Decision | Choice | Trade-off accepted |
| --- | --- | --- | --- |
| 1 | MDM style | Centralized for new items (created in STEP via onboarding); consolidation for legacy ERP items during migration | Two operating modes for a period; requires clear cutover per source |
| 2 | Integration backbone | Kafka between STEP and every system, no point-to-point links | More moving parts than direct integration; gains replay, decoupling and one place to monitor |
| 3 | Canonical model | One versioned product schema at the STEP boundary | Upfront modelling effort; avoids STEP configuration per source format |
| 4 | Read scaling | Separate product data service for high-volume reads | Eventual consistency of a few seconds; protects STEP performance |
| 5 | Hosting | Vendor-hosted STEP SaaS | Less control over upgrades and infrastructure; less operational burden |
| 6 | Write-back | Golden data pushed to ERPs by adapters, not by STEP directly | Adapters to maintain; keeps ERP-specific logic out of STEP |

## 8. Risks and open questions

The largest risks are organizational and migration-related rather than technical.

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Attribute ownership not agreed across teams | Conflicting edits, stalled workflows | Governance forum signs off the ownership table before build |
| Legacy duplicates larger than expected | Long steward queues at cutover | Profile and deduplicate each source before migration |
| Inbound event volume exceeds what one STEP endpoint processes | Growing lag at peak | Load test early; split by topic or batch low-priority changes |
| STEP configuration drift between environments | Failed or risky releases | Single promotion path; no manual changes in production |
| ERP write-back failures go unnoticed | ERPs serve stale data | Reconciliation job and alerts on adapter dead letters |

**Open questions**

- [ ] Does the supplier portal become the only entry point for new items, or can category teams still create items directly in STEP?
- [ ] What latency do ERP order processes actually need for new or changed items?
- [ ] Which STEP version and hosting option is assumed, and what does it support for Kafka throughput and authentication?

**Sources**

- [Stibo STEP documentation: Kafka Receiver (inbound)](https://service.stibosystems.com/documentation/step2024q3/content/dataexc/iiep/creatingiiep/iiepreceivermethods/kafkarec.html)
- [Stibo STEP documentation: Kafka Delivery method (outbound)](https://service.stibosystems.com/documentation/latest/content/dataexc/oiep/creating_an_oiep/oiepdeliverymethods/kafkadelmth.html)
