# Product MDM on Stibo STEP — Reference Architecture

*Version 0.2 · October 2026*

## 1. Context and goals

FreshLine Foods, a fictional food distributor, adopts Stibo STEP as its product MDM platform so that every system works from one governed product record. This document is the production target architecture: it decides what each layer owns and how they connect, not how STEP is configured internally.

**Business problem.** Product data is created in places that disagree: a legacy ERP, an ERP inherited through acquisition, and supplier submissions that arrive as spreadsheets and email. Duplicates, conflicting attributes and slow new-item setup delay sales and cause ordering errors.

**Goals**

- One trusted product record per item, with clear ownership of each attribute
- New item setup governed end to end, from supplier submission to ERP availability
- Product changes available to every consuming system within minutes
- Measurable data quality, reported by source and by attribute group

**Scope:** the product domain, its integrations and the production platform around STEP. **Out of scope:** STEP data model and business rule design, customer and supplier domains, and commercial considerations such as licensing.

**Note on sources.** The design uses only public Stibo documentation and a fictional company. Nothing is taken from any employer's implementation.

## 2. Architecture at a glance

No external system talks to STEP's core directly. ERPs and downstream consumers connect through the Kafka backbone, applications read golden data through STEP Data as a Service (DaaS), and suppliers work inside STEP's own supplier portal. STEP can therefore be upgraded or scaled without touching the systems around it.

See [the context diagram](diagrams/context.md).

ERPs publish changes in and receive golden data back through adapters on the backbone. Suppliers submit and maintain items in the supplier portal. STEP masters and governs. Consumers subscribe to golden events or query DaaS.

## 3. Building blocks

STEP owns mastering, governance and supplier collaboration. Everything around it is deliberately thin, standard and replaceable, so STEP is never coupled directly to an ERP or consumer.

| Layer | Responsibility | Technology (proposed) |
| --- | --- | --- |
| Source systems | Create and change local item data; consume golden data back | ERP-A, ERP-B |
| Supplier onboarding | Supplier self-service entry and maintenance of their items, inside STEP workflows | Stibo STEP supplier portal (Supplier Web UI) |
| Integration layer | Transport, routing, format translation, retries, replay | Managed Kafka with schema registry; integration services on containers |
| MDM platform | Data model, matching and merge, golden record, business rules, stewardship workflows, data quality | Stibo STEP (vendor SaaS preferred) |
| Product data access | Low-latency, on-demand read access to golden records | Stibo STEP Data as a Service (GraphQL APIs) |
| API gateway | Secure, throttled access to DaaS for internal and partner consumers | Any enterprise API gateway |
| Analytical warehouse | History, data quality trends, KPIs, reporting | Cloud data warehouse, loaded from golden and DQ events on Kafka |
| Platform services | Identity, secrets, monitoring, CI/CD | Corporate IdP (SSO), secrets manager, monitoring platform, CI/CD pipeline |

## 4. Integration patterns

Kafka is the backbone in both directions, and STEP connects to it natively: inbound integration endpoints can read from a topic with the [Kafka Receiver](https://service.stibosystems.com/documentation/step2024q3/content/dataexc/iiep/creatingiiep/iiepreceivermethods/kafkarec.html), and outbound endpoints can publish with the [Kafka Delivery method](https://service.stibosystems.com/documentation/latest/content/dataexc/oiep/creating_an_oiep/oiepdeliverymethods/kafkadelmth.html). For on-demand reads, [DaaS](https://service.stibosystems.com/documentation/latest/content/daas/daas.html) receives data published from STEP through an event processor and exposes it through GraphQL APIs.

| Flow | Pattern | Notes |
| --- | --- | --- |
| ERP item changes into STEP | ERP change capture → canonical event on Kafka → STEP inbound endpoint | Integration layer maps each ERP format to one canonical product schema before STEP sees it |
| Supplier new-item submission | Supplier enters item in the STEP supplier portal → STEP onboarding workflow | No integration to build; suppliers see and edit only their own items |
| Golden record distribution | STEP outbound endpoint → Kafka golden topic → each consumer subscribes | One publication, many consumers; consumers never call STEP for bulk data |
| ERP enrichment write-back | Golden topic → per-ERP adapter → ERP API or staging tables | Adapters own each ERP's format, keys and error handling |
| Synchronous lookup | STEP → DaaS; consumer → API gateway → DaaS GraphQL | Shields STEP's core from high-volume reads |
| Analytics | Golden and DQ topics → analytical warehouse | Full change history for KPIs, DQ trends and audit |

**Design rules**

- STEP exchanges one canonical, versioned schema with ERPs; source-specific formats stop at the integration layer.
- Every message carries a source key and an event ID, so consumers can deduplicate and replay safely.
- Failed messages go to dead-letter topics with the error attached and are replayable after a fix.
- Bulk loads (initial migration, large supplier catalogs) use file-based import, not the event path.

## 5. Governance and stewardship

Every attribute group has one owning team and one system of entry, so conflicts are resolved by rule rather than by whoever wrote last.

| Attribute group | System of entry | Owner |
| --- | --- | --- |
| Identity (GTIN, brand, supplier item number) | STEP supplier portal | Supplier onboarding team |
| Descriptions and marketing content | STEP | Product content team |
| Logistics (dimensions, pack, pallet) | STEP supplier portal, verified by stewards | Supply chain data team |
| Commercial (category, status, sourcing) | STEP | Category management |
| ERP-specific settings | Each ERP | ERP operations |

**Stewardship in STEP**

- New items follow a STEP workflow: supplier submission in the supplier portal, validation, enrichment, approval, then publication to ERPs.
- Probable duplicates go to a steward review task instead of merging automatically.
- Business rules block publication of records that fail mandatory checks, and give suppliers feedback in the portal.

**Data quality**

- Rules are defined once, in STEP, and their results are published with each record.
- The analytical warehouse holds DQ results over time, reported by source system, supplier and attribute group.
- A data governance forum reviews DQ trends and rule changes monthly.

## 6. Production readiness

The platform is run like any tier-1 system: separate environments, automated releases, least-privilege access and monitoring that follows a product change end to end. Targets below are proposals to agree with the business.

| Area | Approach |
| --- | --- |
| Environments | Dev, test, pre-production and production for STEP, DaaS and the integration layer; pre-production mirrors production volumes |
| Release management | STEP configuration promoted between environments through a controlled export and import process; integration services deployed through CI/CD with automated contract tests |
| Security | SSO for all internal STEP users with role-based access; suppliers sign in to the supplier portal with supplier-scoped accounts and can only see and edit their own items; internal systems and DaaS consumers authenticate with OAuth client credentials or mTLS through the gateway; encryption in transit and at rest; secrets in a vault, never in configuration files |
| Availability | Target 99.9% for STEP, DaaS and the integration layer during business hours; Kafka retention lets STEP catch up after an outage without data loss |
| Disaster recovery | Proposed targets: RPO 1 hour, RTO 4 hours; tested twice a year |
| Performance | Golden record change visible to consumers within 5 minutes at p95; lookups served from DaaS, not STEP's core |
| Observability | Integration logs, Kafka consumer lag and DaaS publication status in one monitoring view; alerts on dead-letter growth, endpoint failures and workflow backlogs |
| Data migration | Phased cutover by source system; initial loads by bulk import, then switch to events; reconciliation reports before each cutover |
| Support model | Integration and platform teams on call; data stewards own workflow queues and supplier support; vendor support for STEP and DaaS platform issues |

## 7. Key decisions and trade-offs

Seven decisions shape this architecture; each is recorded as an ADR with the options considered.

| # | Decision | Choice | Trade-off accepted |
| --- | --- | --- | --- |
| 1 | MDM style | Centralized for new items (created in STEP via supplier onboarding); consolidation for legacy ERP items during migration | Two operating modes for a period; requires clear cutover per source |
| 2 | Integration backbone | Kafka between STEP and every ERP and bulk consumer, no point-to-point links | More moving parts than direct integration; gains replay, decoupling and one place to monitor |
| 3 | Canonical model | One versioned product schema at the STEP boundary | Upfront modelling effort; avoids STEP configuration per source format |
| 4 | Read scaling | Stibo DaaS for on-demand, high-volume reads | GraphQL-only interface and eventual consistency; no custom read service to build or run; protects STEP's core |
| 5 | Hosting | Vendor-hosted STEP SaaS | Less control over upgrades and infrastructure; less operational burden |
| 6 | Write-back | Golden data pushed to ERPs by adapters, not by STEP directly | Adapters to maintain; keeps ERP-specific logic out of STEP |
| 7 | Supplier onboarding | STEP supplier portal instead of a custom portal | Less control over supplier experience; no portal or integration to build, and supplier edits go straight into governed workflows |

## 8. Risks and open questions

The largest risks are organizational and migration-related rather than technical.

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Attribute ownership not agreed across teams | Conflicting edits, stalled workflows | Governance forum signs off the ownership table before build |
| Legacy duplicates larger than expected | Long steward queues at cutover | Profile and deduplicate each source before migration |
| Inbound event volume exceeds what one STEP endpoint processes | Growing lag at peak | Load test early; split by topic or batch low-priority changes |
| STEP configuration drift between environments | Failed or risky releases | Single promotion path; no manual changes in production |
| ERP write-back failures go unnoticed | ERPs serve stale data | Reconciliation job and alerts on adapter dead letters |
| DaaS query model doesn't fit a consumer's access pattern | Workarounds or extra load on STEP | Validate key consumer queries against DaaS early; consumers needing bulk data subscribe to the golden topic |
| Low supplier adoption of the portal | Submissions keep arriving by email and spreadsheet | Supplier onboarding program, templates for bulk upload, and a cut-off date for off-portal submissions |

**Open questions**

- [ ] Does the supplier portal become the only entry point for new items, or can category teams still create items directly in STEP?
- [ ] What latency do ERP order processes actually need for new or changed items?
- [ ] Which STEP version and hosting option is assumed, and what does it include for Kafka integration, DaaS and the supplier portal?

**Sources**

- [Stibo STEP documentation: Kafka Receiver (inbound)](https://service.stibosystems.com/documentation/step2024q3/content/dataexc/iiep/creatingiiep/iiepreceivermethods/kafkarec.html)
- [Stibo STEP documentation: Kafka Delivery method (outbound)](https://service.stibosystems.com/documentation/latest/content/dataexc/oiep/creating_an_oiep/oiepdeliverymethods/kafkadelmth.html)
- [Stibo STEP documentation: Data as a Service](https://service.stibosystems.com/documentation/latest/content/daas/daas.html)
- [Stibo STEP documentation: Supplier Web UI user guide (PDF)](https://doc.stibosystems.com/doc/version/2025.4/pdf/SupplierWebUIUserGuide.pdf)
