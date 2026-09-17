# 11 — Invariants

> **Purpose:** One place listing every rule that must always hold, grouped by domain, each marked with its certainty. This document will become the backbone of validation logic, tests and acceptance criteria.
> **Rule for this document:** assumptions are never promoted to CONFIRMED. If the brief did not state it, it is PROPOSED or OPEN.

## Markers

| Marker | Meaning |
|---|---|
| **CONFIRMED** | Stated as an architectural constraint. Implementation must enforce it. |
| **PROPOSED** | Reasonable default consistent with the confirmed constraints. Enforce unless the design session overrides it; record any override. |
| **OPEN** | A rule is needed here but its content is undecided. Do not implement a silent default; see the linked open question. |

---

## Item

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-ITEM-01 | A canonical `item_id` identifies exactly one item, and is never reused for another item. | **CONFIRMED** | |
| INV-ITEM-02 | Application workflow state (e.g. `awaiting_photography`, `listing_state`, task/UI state) is never stored as canonical item state. | **CONFIRMED** | The API must offer no generic "set arbitrary attribute" path that would allow it. |
| INV-ITEM-03 | Every change to canonical item state passes through the Item Domain's command boundary. | **CONFIRMED** | Corollary of INV-INT-01. |
| INV-ITEM-04 | Every item has a `version`; every committed change to the item increments it. | **CONFIRMED** | What counts as "a change to the item" is INV-ITEM-07 / `OQ-ITEM-05`. |
| INV-ITEM-05 | `item_id` is immutable and opaque (carries no business meaning). | **PROPOSED** | Format: `OQ-ID-07`. |
| INV-ITEM-06 | An item's `item_type` belongs to the item's `category`. | **PROPOSED** | Redundancy of `item.category_id`: `OQ-CLS-05`. |
| INV-ITEM-07 | Identifier and property changes are changes to the item aggregate and increment `item.version`. | **PROPOSED** | `OQ-ITEM-05`. |
| INV-ITEM-08 | An item has exactly one classification (one category, one item type) at any time. | **PROPOSED** | Multi-classification is not in the brief. |
| INV-ITEM-09 | Whether an item may exist without classification. | **OPEN** | `OQ-ITEM-06`. |
| INV-ITEM-10 | Whether items can be deleted/retired/merged, and what happens to references and identifiers when they are. | **OPEN** | `OQ-ITEM-04`. |
| INV-ITEM-11 | Item never holds quantity or location. | **CONFIRMED** | Those are Inventory facts. |

## Identifiers

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-ID-01 | External/business identifiers (article number, SKU, Shopify IDs, supplier article number, legacy IDs) are never the canonical identity. | **CONFIRMED** | |
| INV-ID-02 | An `ItemIdentifier` record points at exactly one item. | **CONFIRMED** | Whether the same *triple* can appear on two records for two items is INV-ID-06. |
| INV-ID-03 | An identifier resolves according to its `(namespace, identifier_type)` semantics; a bare value is never resolved without them. | **CONFIRMED** | |
| INV-ID-04 | Within a `(namespace, identifier_type)`, a `value` resolves to at most one item. | **PROPOSED** | Default uniqueness rule; may be relaxed per namespace. `OQ-ID-01`, `OQ-ID-03`. |
| INV-ID-05 | Identifiers are added/removed only through Item Domain commands, which emit `ItemIdentifierAdded` / `ItemIdentifierRemoved`. | **PROPOSED** | |
| INV-ID-06 | Exact uniqueness scope: whether the triple is globally unique, and whether an item may carry several identifiers of the same `(namespace, identifier_type)`. | **OPEN** | `OQ-ID-01`, `OQ-ID-02`. |
| INV-ID-07 | Namespaces and identifier types come from a governed registry. | **PROPOSED** | `OQ-ID-04`. |
| INV-ID-08 | Values are normalized per namespace/type before uniqueness is evaluated. | **OPEN** | `OQ-ID-09`. |
| INV-ID-09 | The canonical `item_id` is never registered as an `ItemIdentifier`. | **PROPOSED** | Keeps the two concepts distinct. |

## Classification and properties

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-CLS-01 | Properties assigned to an item must conform to the property definitions allowed for the item's classification. | **CONFIRMED** | |
| INV-CLS-02 | A property value must satisfy its definition's `data_type` and `constraints`. | **CONFIRMED** | |
| INV-CLS-03 | An `ItemType` belongs to exactly one `Category`. | **PROPOSED** | |
| INV-CLS-04 | An item carries at most one value per `PropertyDefinition`. | **PROPOSED** | Multi-valued: `OQ-PROP-02`. |
| INV-CLS-05 | Required properties must be present for an item to be *complete*; whether incompleteness blocks a command or is only reportable is undecided. | **PROPOSED / OPEN** | `OQ-PROP-03`, `OQ-ITEM-06`. |
| INV-CLS-06 | A `PropertyDefinition` in use by items cannot be deleted; it is deprecated instead. | **PROPOSED** | |
| INV-CLS-07 | Reference data (`Category`, `ItemType`, `PropertyDefinition`) is changed only through Item Domain commands. | **CONFIRMED** | Corollary of INV-INT-01. Governance: `OQ-CLS-02`. |
| INV-CLS-08 | Behaviour when reclassification makes existing properties illegal. | **OPEN** | `OQ-CLS-03`. |
| INV-CLS-09 | Precedence when a property is defined at both category and item-type scope. | **OPEN** | `OQ-PROP-01`. |
| INV-CLS-10 | Whether properties without a definition are ever permitted. | **OPEN** | `OQ-PROP-04`. Proposed answer: no. |

## Inventory

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-INV-01 | Inventory references items and locations by canonical ID only; it never defines or stores what an item or location *is*. | **CONFIRMED** | |
| INV-INV-02 | Every inventory change has a traceable business cause: a movement with `movement_type`, `source_application` and `reference_id`. | **CONFIRMED** | |
| INV-INV-03 | Inventory is changed by recording movements, never by overwriting a quantity. | **CONFIRMED** | `adjust` is itself a movement with a reason. |
| INV-INV-04 | Movements are immutable once recorded; corrections are new movements. | **PROPOSED** | |
| INV-INV-05 | Position quantities are always consistent with the sum of movements. | **PROPOSED** | Stored vs derived: `OQ-INV-09`. |
| INV-INV-06 | A movement's `item_id` and `location_id`s refer to existing canonical entities. | **PROPOSED** | Verification mechanism: `OQ-INV-06`. |
| INV-INV-07 | Movement quantity is positive; direction is expressed by from/to. | **PROPOSED** | |
| INV-INV-08 | Whether a position may become negative. | **OPEN** | `OQ-INV-02`. |
| INV-INV-09 | Quantity semantics (arbitrary counts vs 0/1 unique units). | **OPEN** | `OQ-ITEM-01`, `OQ-INV-01`. |
| INV-INV-10 | `adjust` requires an explicit reason. | **PROPOSED** | |
| INV-INV-11 | Inventory holds no item description and no workflow state. | **CONFIRMED** | |

## Location

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-LOC-01 | Location identity is defined only by the Location Domain. | **CONFIRMED** | |
| INV-LOC-02 | `location_id` is stable and immutable. | **PROPOSED** | |
| INV-LOC-03 | If a hierarchy is adopted, it is acyclic. | **PROPOSED** | `OQ-LOC-01`. |
| INV-LOC-04 | A location is never a party; a party is never a location. | **CONFIRMED** | Mirrors INV-PARTY-02. |
| INV-LOC-05 | Which location types may hold stock. | **OPEN** | `OQ-LOC-04`. |
| INV-LOC-06 | Location holds no inventory data. | **CONFIRMED** | |

## Party

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-PARTY-01 | Party identity is defined only by the Party Domain. | **CONFIRMED** | |
| INV-PARTY-02 | A party is not a location and a location is not a party. | **CONFIRMED** | |
| INV-PARTY-03 | `party_id` is stable and immutable. | **PROPOSED** | |
| INV-PARTY-04 | Party does not hold CRM data (history, communication, pipeline). | **CONFIRMED** | "Do not invent a CRM." |
| INV-PARTY-05 | Party references nothing in Item, Inventory or Location. | **PROPOSED** | Party is a leaf. |

## Integration and data ownership

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-INT-01 | Applications cannot directly mutate centralized-domain persistence; all writes go through the domain API. | **CONFIRMED** | |
| INV-INT-02 | Application projections are never authoritative; on disagreement the domain is right. | **CONFIRMED** | |
| INV-INT-03 | Event consumers tolerate at-least-once delivery (idempotent apply). | **CONFIRMED** | |
| INV-INT-04 | Domain events are published only for committed changes, via a transactional outbox. | **CONFIRMED** | |
| INV-INT-05 | Applications reference canonical entities by canonical ID, not by an external identifier used as a foreign key. | **PROPOSED** | During migration, legacy IDs may coexist but must be bridged via `ItemIdentifier`. |
| INV-INT-06 | Applications do not read centralized-domain persistence directly. | **PROPOSED** | `OQ-API-05`. |
| INV-INT-07 | Domains never depend on application state (they are authoritative; application state is workflow). They communicate outward by publishing events, not by calling applications. | **CONFIRMED** (no dependency on app state) / **PROPOSED** (events rather than calls) | The brief shows only the event direction; "never call applications" is derived, not stated. |
| INV-INT-08 | Every committed change eventually produces its event(s). | **CONFIRMED** | Outbox is durable and retried. |
| INV-INT-09 | Per-aggregate event ordering (by version). | **PROPOSED** | `OQ-EVT-03`. |

## Versioning, concurrency and idempotency

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-CC-01 | An update that states an expected version is rejected if the current version differs. | **CONFIRMED** | |
| INV-CC-02 | A successful update increments the version (17 → 18). | **CONFIRMED** | |
| INV-CC-03 | The domain never auto-merges conflicting updates. | **PROPOSED** | Caller re-reads and decides. |
| INV-CC-04 | A retried command with the same idempotency key does not re-execute; the original outcome is returned. | **PROPOSED** (mechanism) / **CONFIRMED** (goal) | Brief: "retries do not accidentally duplicate operations". |
| INV-CC-05 | Same idempotency key with different payload is rejected. | **PROPOSED** | `OQ-CC-02`. |
| INV-CC-06 | Whether `expected_version` is mandatory on all updates. | **OPEN** | `OQ-CC-01`. |
| INV-CC-07 | Every event carries the resulting aggregate version. | **PROPOSED** | Needed for INV-INT-03 to be implementable cleanly. |

---

## Cross-domain consistency checks (for reviewers)

These are not invariants of a single domain; they are checks that the set of invariants above is coherent.

| Check | Result |
|---|---|
| Item vs Inventory ownership | INV-ITEM-11 and INV-INV-11 are complementary: Item has no quantity/location; Inventory has no description. No overlap. |
| Item vs application workflow | INV-ITEM-02 (Item side) and INV-INT-02 (application side) together forbid workflow state in the domain and canonical authority in the app. |
| Location vs Party | INV-LOC-04 / INV-PARTY-02 are the same rule stated from both sides. Location → Party reference is PROPOSED, never the reverse (INV-PARTY-05). |
| Canonical ID vs external identifiers | INV-ID-01, INV-ID-09 and INV-INT-05 together: external identifiers point at `item_id`, are never it, and are never used as foreign keys by applications post-migration. |
| Canonical state vs projections | INV-INT-02 + INV-INT-03 + INV-CC-07: projections are caches, applied idempotently by version. |
| Commands vs direct writes | INV-INT-01 + INV-ITEM-03 + INV-CLS-07: every write, including reference data, goes through commands. |
| Confirmed vs assumed | Every CONFIRMED row above traces to an explicit statement in the brief. Everything derived is PROPOSED; everything undecided is OPEN. |
