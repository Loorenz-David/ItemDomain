# 12 — Open Questions

> **Purpose:** Every architectural decision that still has to be made, discovered while formalizing the architecture. Nothing here is answered; where the documents propose a default, it is labelled as a proposal and the question stays open until explicitly decided.
>
> **Timing legend:** **Before implementation** = must be decided before the affected component is built, because the answer changes its shape. **Deferrable** = can be decided later without rework, provided the initial design does not foreclose it.

Each entry: **Decision** · **Why it matters** · **Affects** · **Timing**.

---

## Item

### OQ-ITEM-01 — Item granularity: product definition or physical unit?
- **Decision:** Is an `Item` a *product/SKU-level definition* that can exist in quantity > 1 ("Stockholm Sofa, 3-seat, leather"), a *unique physical unit* (this particular sofa, quantity always 0 or 1), or does the model need both (e.g. a definition and serialized units)?
- **Why it matters:** The brief's examples pull both ways: Shopify variant IDs and SKUs are product-level; a workflow like "awaiting photography" for a specific item, and dimensions measured by a worker, suggest unique units. The answer determines whether `InventoryPosition.quantity` is a count or a presence flag, whether one supplier article number maps to one or many items, and how Shopify variants map to items.
- **Affects:** Item model, identifier uniqueness (OQ-ID-03), Inventory quantity semantics (OQ-INV-01), Shopify integration, migration.
- **Timing:** **Before implementation.** This is the single most shape-changing question.

### OQ-ITEM-02 — Is `name` a first-class canonical attribute?
- **Decision:** The brief lists `item.name` under "canonical" and shows `"name": "Stockholm Sofa"` in a projection, but the conceptual `Item` model has no `name`. Is it a first-class attribute, a property definition, or not canonical at all? Same for `description`.
- **Why it matters:** Every application displays a name. If it is not canonical, each app keeps its own — exactly the divergence the architecture is meant to end. If it *is* canonical, is a Shopify listing title the same thing or channel copy (OQ-PROP-06)?
- **Affects:** Item model, events, projections, Seller/Shopify integration.
- **Timing:** **Before implementation.**

### OQ-ITEM-03 — Dimensions and weight: first-class or properties? Units?
- **Decision:** Are `dimensions` and `weight` universal first-class attributes (as modelled) or classification-driven properties like everything else? What units, and is the unit fixed or stored per value? Are they optional?
- **Why it matters:** Making them first-class asserts *every* item has meaningful dimensions. If some item types don't, they become nullable noise. Unit ambiguity causes silent data corruption.
- **Affects:** Item model, validation, projections.
- **Timing:** **Before implementation** (schema).

### OQ-ITEM-04 — Item lifecycle: deletion, retirement, merging, canonical status
- **Decision:** Can items be deleted? Retired/archived? Merged when two records turn out to be the same thing (likely during migration)? Is there a *canonical* lifecycle status (e.g. `active` / `retired` / `merged_into`) — which would be legitimate because it is a fact about the record, not a workflow?
- **Why it matters:** Deleting an item referenced by Inventory positions, application records and identifiers has consequences everywhere. Merge requires identifier reassignment (OQ-ID-06) and a redirect for old IDs. A canonical status must be carefully distinguished from workflow status to avoid re-opening the door to `awaiting_photography`.
- **Affects:** Item commands/events, Inventory, all applications, migration.
- **Timing:** **Before implementation** for at least "can items be deleted?"; merge semantics may be deferrable if migration does not need them.

### OQ-ITEM-05 — Aggregate and version boundary
- **Decision:** Do identifier and property changes increment `item.version` (single aggregate), or are they versioned separately?
- **Why it matters:** Trade-off between contention (every app editing anything on an item conflicts with every other) and consistency (classification/property validity can race). See [09](09-versioning-concurrency-and-idempotency.md#aggregate-boundary-trade-off-oq-item-05).
- **Affects:** Command semantics, event stream(s), projection logic, Inventory analogue.
- **Timing:** **Before implementation.**

### OQ-ITEM-06 — Mandatory fields at creation; unclassified items
- **Decision:** What must be present to create an item? Can an item exist without category/type (e.g. a scanner creates a stub from a barcode before anyone classifies it)? Are required properties enforced at creation, at classification, or only reported as "incomplete"?
- **Why it matters:** Determines whether "create stub now, enrich later" workflows are possible, and whether INV-CLS-05 is a hard rule or a completeness report.
- **Affects:** `CreateItem`, validation, Scanner/Worker flows.
- **Timing:** **Before implementation.**

### OQ-ITEM-07 — Item-to-item relationships
- **Decision:** Are there variants (parent product / child variants), sets/bundles, components, or "same model as" relationships?
- **Why it matters:** Shopify has product → variants natively. If items are unique units, "same model" grouping may be wanted for reuse of properties.
- **Affects:** Item model, Shopify mapping.
- **Timing:** **Deferrable**, provided OQ-ITEM-01 is answered first.

### OQ-ITEM-08 — Images / media
- **Decision:** Are photos canonical item data, application (Worker/Seller) data, or a separate concern?
- **Why it matters:** Photography is clearly a workflow; the *resulting photos* may be a canonical fact about the item. Not stated in the brief.
- **Affects:** Item model or a future domain; Worker/Seller.
- **Timing:** **Deferrable**; must simply not be assumed canonical in phase 1.

## Identity

### OQ-ID-01 — Uniqueness scope of identifiers
- **Decision:** Is `(namespace, identifier_type, value)` unique across all items? Per namespace only? Never enforced? Configurable per namespace/type?
- **Why it matters:** Resolution is only unambiguous if uniqueness holds. Without it `ResolveIdentifier` must return sets, and Scanner/integration flows need a disambiguation step.
- **Affects:** Identifier model, `AddItemIdentifier` validation, `ResolveIdentifier`, migration.
- **Timing:** **Before implementation.**

### OQ-ID-02 — Multiple identifiers of the same type on one item
- **Decision:** May an item carry two `shopify/variant_id`s (e.g. listed twice), two article numbers (old and new)?
- **Why it matters:** Projections that flatten "the article number" need a rule for choosing; integrations need to know which one is current.
- **Affects:** Identifier model, projections, Shopify sync.
- **Timing:** **Before implementation.**

### OQ-ID-03 — Same identifier on multiple items
- **Decision:** Can one `supplier/article_number/X` legitimately be attached to many items (many physical units of the same supplier article)?
- **Why it matters:** Directly follows from OQ-ITEM-01. If items are units, shared supplier/product identifiers are the norm and INV-ID-04 must be per-namespace.
- **Affects:** Same as OQ-ID-01.
- **Timing:** **Before implementation.**

### OQ-ID-04 — Governed namespace/type registry vs free-form
- **Decision:** Is the set of namespaces and identifier types a closed list managed in the Item Domain, or can callers invent them?
- **Why it matters:** Uniqueness and normalization rules are per namespace/type; they cannot be enforced for unknown ones.
- **Affects:** Identifier validation, reference data, authorization.
- **Timing:** **Before implementation** (proposed: governed).

### OQ-ID-05 — Meaning of `source` and content of `metadata`
- **Decision:** What does `source` capture that `namespace` does not (e.g. "which import job / which application registered this")? What may go in `metadata`?
- **Why it matters:** Undefined fields become dumping grounds.
- **Affects:** Identifier model.
- **Timing:** **Before implementation** (small).

### OQ-ID-06 — Identifier lifecycle
- **Decision:** On `RemoveItemIdentifier`, is history retained (so an old Shopify ID can still be traced)? Can an identifier be reassigned from one item to another (needed for merges)?
- **Why it matters:** Traceability during and after migration.
- **Affects:** Identifier model, events.
- **Timing:** **Deferrable** unless merges are needed for migration.

### OQ-ID-07 — `item_id` format
- **Decision:** UUID, prefixed string (`itm_…`), other? Generated by the domain only?
- **Why it matters:** Appears in every application, label, URL.
- **Affects:** Everything that stores IDs.
- **Timing:** **Before implementation.**

### OQ-ID-08 — Does the Item Domain generate business identifiers?
- **Decision:** Does the domain *issue* internal article numbers / SKUs, or only *record* identifiers issued elsewhere?
- **Why it matters:** Issuing implies sequences, formats and uniqueness guarantees the domain must own; recording does not.
- **Affects:** Identifier commands, migration.
- **Timing:** **Before implementation.**

### OQ-ID-09 — Value normalization
- **Decision:** Case sensitivity, whitespace, leading zeros, per namespace/type.
- **Why it matters:** Uniqueness cannot be enforced reliably without it; scanner input is noisy.
- **Affects:** Identifier validation and resolution.
- **Timing:** **Before implementation.**

## Classification

### OQ-CLS-01 — Hierarchy depth
- **Decision:** Fixed two levels (Category → ItemType) or deeper/variable?
- **Why it matters:** Property scoping rules multiply with depth.
- **Affects:** Reference-data model, `allowed(item)` computation.
- **Timing:** **Deferrable** if two levels are adopted now and the model does not forbid a parent on Category later.

### OQ-CLS-02 — Reference-data governance
- **Decision:** Who may create/change categories, item types and property definitions? Through which application (Manager?) and with what review?
- **Why it matters:** Reference-data changes affect every item of a classification and every application's forms. Uncontrolled changes are a silent breaking change.
- **Affects:** Authorization, Manager app, events.
- **Timing:** **Before implementation.**

### OQ-CLS-03 — Reclassification semantics
- **Decision:** When an item's type changes and some properties are no longer allowed: reject, drop, keep-but-flag, or require the command to specify?
- **Why it matters:** Data loss vs invalid state vs command complexity.
- **Affects:** `ClassifyItem`, validation, events.
- **Timing:** **Before implementation.**

### OQ-CLS-04 — Unclassified items
- See **OQ-ITEM-06**.

### OQ-CLS-05 — Is `item.category_id` redundant?
- **Decision:** Keep both `category_id` and `item_type_id` on the item (with INV-ITEM-06 enforcing agreement), or derive category from type?
- **Why it matters:** Redundant data can disagree; derived data cannot.
- **Affects:** Item model.
- **Timing:** **Before implementation** (small).

## Properties

### OQ-PROP-01 — Scope precedence
- **Decision:** When a property is defined at category level and again at item-type level, which wins? Is the item-type definition a refinement (narrower constraints) or an override?
- **Affects:** `allowed(item)`, validation.
- **Timing:** **Before implementation.**

### OQ-PROP-02 — Data types, enums, multi-valued properties
- **Decision:** Supported `data_type` set; how enumerations are defined; whether a property may hold multiple values (e.g. several materials).
- **Affects:** `PropertyDefinition`, validation, projections.
- **Timing:** **Before implementation.**

### OQ-PROP-03 — Definition evolution and enforcement timing
- **Decision:** What happens to existing items when a definition changes (new `required`, tightened constraint)? Are definitions versioned? When is `required` enforced?
- **Why it matters:** Otherwise a reference-data change can make thousands of items invalid with no path to fix them.
- **Affects:** Reference-data commands, validation, reporting.
- **Timing:** **Before implementation** (at least a policy).

### OQ-PROP-04 — Free-form properties
- **Decision:** May an item carry a property with no definition?
- **Why it matters:** Convenient, but re-creates the "anything goes" problem. Proposed: no.
- **Timing:** **Before implementation.**

### OQ-PROP-05 — Localization
- **Decision:** Are property names / enum values / category names localized?
- **Timing:** **Deferrable**.

### OQ-PROP-06 — Canonical descriptions vs channel copy
- **Decision:** Which descriptive text is a canonical fact (a neutral name/description) and which is channel/campaign copy owned by Seller/Shopify?
- **Why it matters:** Directly linked to OQ-ITEM-02; prevents Seller from either duplicating or overwriting canonical text.
- **Timing:** **Before implementation** for the initial property set.

## Inventory

### OQ-INV-01 — Quantity semantics
- **Decision:** Arbitrary counts, or 0/1 per unique unit (see OQ-ITEM-01)?
- **Timing:** **Before implementation.**

### OQ-INV-02 — Negative inventory
- **Decision:** Allowed (with flag), forbidden, or allowed for specific movement types (e.g. `sell` before `receive` in a rush)?
- **Why it matters:** Determines whether Inventory can *reject* operationally real events or must record them and flag.
- **Timing:** **Before implementation.**

### OQ-INV-03 — Precise operation semantics
- **Decision:** Exact meaning of `receive`, `place`, `transfer`, `remove`, `sell`, `return`, `adjust`; nullability of from/to for each; what `reference_id` points to per type; whether `sell`/`return` target a sink or a location.
- **Timing:** **Before implementation.**

### OQ-INV-04 — Reservations / allocations
- **Decision:** Is "sold but not yet shipped" / "reserved for a customer" an Inventory concern? If so, is it a movement to a logical location or a separate concept?
- **Timing:** **Deferrable** if the initial model does not forbid it; boundary decision needed eventually.

### OQ-INV-05 — Condition / stock state
- **Decision:** Where does "damaged" / "quarantine" / "display model" belong — Item (a fact about the unit), Inventory (a state of stock), a logical location, or an application?
- **Timing:** **Deferrable**, but must not be improvised into Item as a `status`.

### OQ-INV-06 — Validating `item_id` / `location_id` existence
- **Decision:** Synchronous check against Item/Location APIs, a local projection fed by their events, or trust the caller?
- **Why it matters:** Coupling and availability vs consistency.
- **Timing:** **Before implementation.**

### OQ-INV-07 — Idempotency key for movements
- **Decision:** Is `(source_application, reference_id)` sufficient, or is an explicit key required?
- **Timing:** **Before implementation.**

### OQ-INV-08 — Party references and ownership
- **Decision:** Do movements record a `party_id` (supplier on receive, customer on sell)? Is stock ownership/consignment a concept?
- **Timing:** **Deferrable**; tied to OQ-PARTY-01.

### OQ-INV-09 — Stored positions vs derived from movements
- **Decision:** Persistence strategy.
- **Timing:** Design session.

## Location

### OQ-LOC-01 — Hierarchy
- **Decision:** Parent/child locations, and how deep?
- **Timing:** **Before implementation** of Location (minimal: allow optional parent or not).

### OQ-LOC-02 — Location types
- **Decision:** The initial type list and whether it is governed.
- **Timing:** **Before implementation** (minimal list).

### OQ-LOC-03 — Relationship to Party
- **Decision:** Do dealer/customer locations reference a `party_id`?
- **Timing:** **Before implementation** of Location if such locations are in phase 1; otherwise deferrable.

### OQ-LOC-04 — Logical locations and stock-capable types
- **Decision:** Are "in transit", "sold", "lost" locations, or do movements use null from/to? Which types can hold stock?
- **Timing:** Coupled to OQ-INV-03; **before implementation** of Inventory.

### OQ-LOC-05 — Governance
- **Decision:** Who creates locations, via which application; naming conventions to avoid duplicates.
- **Timing:** **Before implementation** (small).

### OQ-LOC-06 — Location identifiers
- **Decision:** Do locations need external identifiers (barcodes, legacy IDs) using the same pattern as items?
- **Timing:** **Deferrable** unless Scanner needs it in phase 1.

## Party

### OQ-PARTY-01 — Is Party needed in phase 1?
- **Decision:** Which centralized-domain references to a party are actually required initially (supplier on item? customer on sell? owner of location?). If none, Party can be deferred entirely.
- **Why it matters:** Avoids building a domain nothing consumes.
- **Timing:** **Before implementation** (as a scoping decision).

### OQ-PARTY-02 — Kinds vs roles
- **Decision:** Model `person`/`organization` as kind and `supplier`/`customer`/`dealer` as roles? Can a party hold several roles? Do roles live on the party or on the referencing relationship?
- **Timing:** **Deferrable** until Party is built.

### OQ-PARTY-03 — External identifiers for parties
- **Decision:** Reuse the `ItemIdentifier` pattern (namespace/type/value) for parties (Shopify customer ID, supplier numbers)?
- **Timing:** **Deferrable.**

### OQ-PARTY-04 — Personal data
- **Decision:** What personal data may be stored, retention, access — before any person record exists.
- **Timing:** **Before implementation** of Party if persons are in scope.

### OQ-PARTY-05 — Application users vs parties
- **Decision:** Are application users (the worker using the Worker app) ever parties? Proposed: no; user identity is an authorization concern.
- **Timing:** **Before implementation** of authorization (OQ-AUTHZ-02).

## API

### OQ-API-01 — Command handling model and protocol
- **Decision:** Synchronous request/response (proposed) vs asynchronous acceptance; transport/protocol.
- **Timing:** Design session; **before implementation**.

### OQ-API-02 — Command granularity
- **Decision:** Fine-grained intent commands, coarse `UpdateItem`, or both.
- **Why it matters:** Drives event taxonomy (OQ-EVT-01) and authorization granularity.
- **Timing:** **Before implementation.**

### OQ-API-03 — Bulk / batch operations
- **Decision:** Needed for migration and imports? Same validation and per-item outcomes?
- **Timing:** **Before implementation** if migration uses the API (it should).

### OQ-API-04 — Query and search capabilities
- **Decision:** Filtering by classification/properties/identifiers; pagination; full-text; whether search is a domain query or an application projection.
- **Timing:** **Before implementation** for the minimum; richer search deferrable.

### OQ-API-05 — Direct read access to domain databases
- **Decision:** Prohibited for applications (proposed); what about reporting/analytics?
- **Timing:** **Before implementation.**

### OQ-API-06 — Error contract
- **Decision:** How validation errors, version conflicts, identifier conflicts, authorization errors and idempotent replays are represented.
- **Timing:** Design session; **before implementation**.

## Events

### OQ-EVT-01 — Final event taxonomy
- **Decision:** Fine-grained only, coarse snapshot only, or both; reference-data and Inventory/Location/Party events.
- **Timing:** **Before implementation.**

### OQ-EVT-02 — Payload shape
- **Decision:** Delta vs full snapshot (vs both). Snapshots simplify consumers and tolerate reordering; deltas are smaller and more expressive.
- **Timing:** **Before implementation.**

### OQ-EVT-03 — Ordering guarantee
- **Decision:** Per-aggregate ordering promised or not.
- **Timing:** **Before implementation** (affects consumer design).

### OQ-EVT-04 — Transport and outbox relay
- **Decision:** Technology.
- **Timing:** Design session.

### OQ-EVT-05 — Replay / backfill
- **Decision:** How a new consumer reaches current state: retained event log, snapshot events on demand, bulk query.
- **Timing:** **Before implementation** of the first projection.

### OQ-EVT-06 — Event schema versioning
- **Timing:** **Before implementation** (at least a convention).

### OQ-EVT-07 — Retention
- **Timing:** Deferrable, coupled to OQ-EVT-05 and OQ-OPS-01.

## Authorization

### OQ-AUTHZ-01 — Which application may issue which commands
- **Decision:** A per-application command allowlist (e.g. Scanner: `transfer` only; Manager: everything).
- **Timing:** **Before implementation.**

### OQ-AUTHZ-02 — Actor identity
- **Decision:** Do commands carry only the application identity, or also the end user? Needed for audit ("who changed the dimensions?").
- **Timing:** **Before implementation.**

### OQ-AUTHZ-03 — Reference-data administration rights
- **Decision:** Who may change classification vocabulary, location types, movement types.
- **Timing:** **Before implementation.**

## Migration

### OQ-MIG-01 — System of record for bootstrap
- **Decision:** Which application's item data seeds the Item Domain when several hold the same item with different values?
- **Timing:** **Before implementation** of migration.

### OQ-MIG-02 — Duplicate detection and merge
- **Decision:** How the same physical thing recorded in multiple apps is recognised (shared identifiers? manual?) and merged.
- **Timing:** **Before implementation** of migration.

### OQ-MIG-03 — Cut-over strategy
- **Decision:** Dual-write period? Strangler pattern per application? Who is authoritative during transition?
- **Timing:** **Before implementation** of migration.

### OQ-MIG-04 — Shopify's role
- **Decision:** Is Shopify a *source* of item facts (products created there flow in), a *consumer* (items flow out), or both — and if both, which wins on conflict?
- **Timing:** **Before implementation** of the Shopify integration.

## Application integration

### OQ-APP-01 — Projection freshness
- **Decision:** Per application: acceptable lag; need for read-your-own-writes after a command.
- **Timing:** **Before implementation** of each application's projection.

### OQ-APP-02 — Query API vs projection on hot paths
- **Decision:** Whether applications may synchronously query the domain in user-facing paths, or must use projections.
- **Timing:** **Before implementation.**

### OQ-APP-03 — Scanner constraints
- **Decision:** Offline operation, resolution latency, queued commands.
- **Timing:** **Before implementation** of Scanner integration.

### OQ-APP-04 — Who creates items
- **Decision:** Manager only? Scanner stubs? Shopify import? Worker on intake?
- **Timing:** **Before implementation.**

## Operational concerns

### OQ-OPS-01 — Audit requirements
- **Decision:** What must be reconstructable (who/what/when/why), retention, and whether the event log serves as the audit log.
- **Timing:** **Before implementation.**

### OQ-OPS-02 — Availability and degraded operation
- **Decision:** What applications do when a domain API is unavailable (queue commands? read-only from projection?).
- **Timing:** **Before implementation** of the first consumer.

### OQ-OPS-03 — Deployment topology
- **Decision:** Four processes, or a modular deployable with four isolated persistence boundaries. (Separate *responsibilities and persistence* are confirmed; separate *processes* are not required by the brief.)
- **Timing:** Design session.

### OQ-OPS-04 — Observability and failure handling
- **Decision:** Dead-lettering, alerting on outbox lag, poison events, metrics.
- **Timing:** **Before go-live**; deferrable for initial build.

### OQ-OPS-05 — Environments and test data
- **Decision:** Reference-data seeding per environment; anonymised item data.
- **Timing:** Deferrable.

## Concurrency (cross-cutting)

### OQ-CC-01 — Is `expected_version` mandatory?
- **Decision:** Mandatory / optional / command-dependent. See [09](09-versioning-concurrency-and-idempotency.md).
- **Timing:** **Before implementation.**

### OQ-CC-02 — Idempotency key contract
- **Decision:** Scope (per application?), retention window, behaviour on payload mismatch.
- **Timing:** **Before implementation.**

### OQ-CC-03 — Inventory concurrency mechanism
- **Decision:** Per-position version, per-item version, or database serialization for movements.
- **Timing:** **Before implementation** of Inventory.

---

## Summary — decisions required before implementation starts

Minimum set that changes the shape of what is built:

| # | Question | Why first |
|---|---|---|
| 1 | OQ-ITEM-01 Item granularity | Everything about identifiers and inventory depends on it. |
| 2 | OQ-ID-01 / 02 / 03 Identifier uniqueness | Resolution semantics and migration matching depend on it. |
| 3 | OQ-ITEM-05 Aggregate/version boundary | Command, event and projection design depend on it. |
| 4 | OQ-ITEM-02 / OQ-PROP-06 `name` and descriptive text | Every projection shows a name. |
| 5 | OQ-ITEM-04 Item lifecycle (at least deletion) | References from Inventory and apps depend on it. |
| 6 | OQ-CLS-02 / OQ-PROP-03 Reference-data governance and evolution | Otherwise vocabulary changes silently break items. |
| 7 | OQ-API-02 / OQ-EVT-01 / OQ-EVT-02 Command granularity, event taxonomy, payload shape | Contracts other teams build against. |
| 8 | OQ-AUTHZ-01 / 02 Who may do what, as whom | Cannot build the write boundary without it. |
| 9 | OQ-MIG-01 / 02 / 04 Seeding, duplicates, Shopify role | The domain is empty without migration. |
| 10 | OQ-INV-01 / 02 / 03 Quantity semantics, negative stock, operation semantics | Inventory cannot be built on candidates. |
