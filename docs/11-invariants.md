# 11 — Invariants

> **Purpose:** One place listing every rule that must always hold, grouped by domain, each marked with its certainty. This document will become the backbone of validation logic, tests and acceptance criteria.
> **Rule for this document:** assumptions are never promoted to CONFIRMED. If the brief did not state it, it is PROPOSED or OPEN.

## Markers

| Marker | Meaning |
|---|---|
| **CONFIRMED** | Stated as an architectural constraint. Implementation must enforce it. |
| **PROPOSED** | Reasonable default consistent with the confirmed constraints. Enforce unless the design session overrides it; record any override. |
| **OPEN** | A rule is needed here but its content is undecided. Do not implement a silent default; see the linked open question. |
| **WITHDRAWN** | Recorded here earlier, then superseded by a later decision. Kept so it is not reintroduced by someone reading an older draft. |

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
| INV-ITEM-07 | Identifier, property **and image** changes are changes to the item aggregate and increment `item.version`. | **CONFIRMED** | `OQ-ITEM-05` resolved — "that is the point" of the version. Contention is accepted deliberately; see [09](09-versioning-concurrency-and-idempotency.md). |
| INV-ITEM-08 | An item has exactly one classification (one category, one item type) at any time. | **PROPOSED** | Multi-classification is not in the brief. |
| INV-ITEM-09 | An item always has a classification; it can never be created or left unclassified. | **CONFIRMED** | `OQ-ITEM-06` resolved. Makes `allowed(item)` always well-defined. |
| INV-ITEM-10 | Items can be deleted, and every deletion is a **soft** delete: the record is retained and flagged, never physically removed. | **CONFIRMED** | `OQ-ITEM-04` resolved. Query visibility of deleted items: `OQ-ITEM-10`. |
| INV-ITEM-11 | Item never holds quantity or location. | **CONFIRMED** | Those are Inventory facts. |
| INV-ITEM-12 | There is no retire and no merge operation on items. Two item records are never fused; "more of this thing" is a quantity fact in Inventory. | **CONFIRMED** | `OQ-ITEM-04` resolved. |
| INV-ITEM-13 | Deleting an item does not consult Inventory; the Item Domain never reads Inventory state. | **CONFIRMED** | Preserves the forbidden `Item → Inventory` direction. Phantom positions are possible by design (`OQ-ITEM-10`). |
| INV-ITEM-14 | An Item is the canonical identity assigned by the business to **one or more** physical units currently treated as the same thing for identification purposes. The number of units under that identity is exclusively an Inventory concern. **There must be no assumption or invariant that one Item equals one physical unit.** | **CONFIRMED** | `OQ-ITEM-01`, `OQ-ITEM-12`. One Item per unique vintage piece is a business usage pattern, not a domain rule. |
| INV-ITEM-20 | Multiple physical units may share one `item_id`, one `article_number` and one `sku`. | **CONFIRMED** | `OQ-ITEM-12`. Follows directly from INV-ITEM-14. |
| INV-ITEM-21 | Identity groupings may later be split or combined by separate "marriage / divorce" business machinery. Its semantics are undefined and out of scope, and it **must not be pre-modelled** in the Item Domain. | **CONFIRMED** (that it is out of scope) | `OQ-ITEM-07`. Reinforces INV-ITEM-15: no relationships, parent/child, sets or variants introduced in anticipation of it. |
| INV-ITEM-22 | Identity grouping is a business decision made at registration and management. The domain records and enforces the resulting identity; it never infers whether two units are similar enough to share one. | **CONFIRMED** | `OQ-ITEM-12`. The criteria a human applies stay outside this specification. |
| INV-ITEM-15 | An Item is an independent canonical row with no relationships to other Items: no variants, sets, components, parent/child or generic item-to-item links. | **CONFIRMED** | `OQ-ITEM-07` resolved. Grouping/splitting identity is separate "marriage / divorce" machinery, out of scope. |
| INV-ITEM-16 | At creation an item must carry `article_number`, `category`, `item_type` and all required properties for that classification — and nothing else is mandatory. | **CONFIRMED** | `OQ-ITEM-06` resolved. `sku`, dimensions and weight arrive later. |
| INV-ITEM-17 | `dimensions` and `weight` are universal first-class attributes (every object has them); properties are conditional on category and type. | **CONFIRMED** | `OQ-ITEM-03` resolved for universality. **Units remain undecided.** |
| INV-ITEM-18 | Soft deletion follows the industry standard: a deletion marker, excluded from queries by default, opt-in to include, restore possible, no cascade to other domains. | **CONFIRMED** | `OQ-ITEM-10`. Does not relax permanent identifier uniqueness (INV-CID-06). |
| INV-ITEM-19 | Neither `name` nor `description` exists. A human recognises a piece by its type, its image and its article number or SKU. | **CONFIRMED** | `OQ-ITEM-02` / `OQ-ITEM-09` resolved. |

## Canonical business identifiers (article number, SKU)

Identifiers issued by **our own** business. The Item Domain can guarantee them, which is what makes them identity.

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-CID-01 | `article_number` and `sku` are canonical identity and are first-class attributes of the Item, not `ItemIdentifier` records. | **CONFIRMED** | |
| INV-CID-02 | Each of `article_number` and `sku` identifies at most one item. | **CONFIRMED** | Follows from being identity: an identifier resolving to two items is not identity. |
| INV-CID-03 | The Item Domain enforces uniqueness and format on supplied values; it does **not** generate them. | **CONFIRMED** | Values are supplied by the creating application or person. |
| INV-CID-04 | `article_number` and `sku` are independent; neither is derived from the other. | **PROPOSED** | They are two distinct attributes (CONFIRMED); what distinguishes them is `OQ-CID-01`. |
| INV-CID-05 | Mutability: whether either may change after assignment, and what happens to history if it does. | **OPEN** | `OQ-CID-04`. `item_id` shields application foreign keys either way. |
| INV-CID-06 | `article_number` and `sku` are permanently unique: a soft-deleted item keeps its values, and they are never reused by another item. | **CONFIRMED** | `OQ-ITEM-04`; resolves half of `OQ-CID-05`. |
| INV-CID-07 | `article_number` is required at creation (assigned at registration); `sku` is absent until the restored piece is published for sale. | **CONFIRMED** | `OQ-CID-01` and `OQ-CID-02` resolved. A missing `sku` is **not** a listing-status flag — see INV-ITEM-02. |
| INV-CID-08 | Format/validation rules exist and are defined by the business. | **OPEN** | `OQ-CID-03`. Until defined, the domain can enforce uniqueness but not format. |
| INV-CID-09 | Value normalization (case, whitespace, leading zeros) before uniqueness is evaluated. | **OPEN** | `OQ-CID-05`. Must be settled before uniqueness can be enforced reliably. |

## External identifiers

Identifiers issued by **foreign** authorities (Shopify, suppliers, systems being replaced). The Item Domain records them; it cannot guarantee them.

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-ID-01 | External identifiers (Shopify product/variant IDs, legacy system IDs, supplier article numbers, other foreign IDs) are never the canonical identity. | **CONFIRMED** | Our own article numbers and SKUs are **not** in this group — see INV-CID-01. |
| INV-ID-02 | An `ItemIdentifier` record points at exactly one item. | **CONFIRMED** | Whether the same *triple* can appear on two records for two items is INV-ID-06. |
| INV-ID-03 | An external identifier resolves according to its `(namespace, identifier_type)` semantics; a bare value is never resolved without them. | **CONFIRMED** | |
| INV-ID-04 | Within a `(namespace, identifier_type)`, a `value` resolves to at most one item. | **PROPOSED** | Default uniqueness rule; may be relaxed per namespace. `OQ-ID-01`, `OQ-ID-03`. |
| INV-ID-05 | External identifiers are added/removed only through Item Domain commands, which emit `ItemIdentifierAdded` / `ItemIdentifierRemoved`. | **PROPOSED** | |
| INV-ID-06 | Exact uniqueness scope: whether the triple is globally unique, and whether an item may carry several identifiers of the same `(namespace, identifier_type)`. | **OPEN** | `OQ-ID-01`, `OQ-ID-02`. |
| INV-ID-07 | Namespaces and identifier types come from a governed registry. | **PROPOSED** | `OQ-ID-04`. |
| INV-ID-08 | Values are normalized per namespace/type before uniqueness is evaluated. | **OPEN** | `OQ-ID-09`. |
| INV-ID-09 | Canonical identities (`item_id`, `article_number`, `sku`) are never registered as `ItemIdentifier`s. | **PROPOSED** | Keeps the two concepts distinct; the identifier table is for foreign-issued values only. |
| INV-ID-10 | Supplier article numbers are external (the supplier issues them). | **PROPOSED** | Inferred from the issuing-authority principle, not explicitly stated. `OQ-ID-10`. |

## Classification and properties

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-CLS-01 | Properties assigned to an item must conform to the property definitions allowed for the item's classification. | **CONFIRMED** | |
| INV-CLS-02 | A property value must satisfy its definition's `data_type` and `constraints`. | **CONFIRMED** | |
| INV-CLS-03 | An `ItemType` belongs to exactly one `Category`. | **PROPOSED** | |
| INV-CLS-04 | An item carries at most one value per `PropertyDefinition`. | **PROPOSED** | Multi-valued: `OQ-PROP-02`. |
| INV-CLS-05 | Required properties must be present **at creation**; an item cannot be created missing one. | **CONFIRMED** | `OQ-ITEM-06` resolved. Settles the "when" half of `OQ-PROP-03` and sharpens the other half — items created before a property became required would be retroactively invalid. |
| INV-CLS-06 | A `PropertyDefinition` in use by items cannot be deleted; it is deprecated instead. | **PROPOSED** | |
| INV-CLS-07 | Reference data (`Category`, `ItemType`, `PropertyDefinition`) is changed only through Item Domain commands. | **CONFIRMED** | Corollary of INV-INT-01. Governance: `OQ-CLS-02`. |
| INV-CLS-08 | Behaviour when reclassification makes existing properties illegal. | **OPEN** | `OQ-CLS-03`. |
| INV-CLS-09 | Precedence when a property is defined at both category and item-type scope. | **OPEN** | `OQ-PROP-01`. |
| INV-CLS-10 | Whether properties without a definition are ever permitted. | **OPEN** | `OQ-PROP-04`. Proposed answer: no. |

## Item images and presentational assets

| ID | Invariant | Status | Notes |
|---|---|---|---|
| INV-IMG-01 | An item's images are canonical: they are part of its visual description and are owned by the Item Domain. | **CONFIRMED** | What the thing looks like is a fact about the thing. |
| INV-IMG-02 | The Item Domain stores absolute URLs plus a storage/source `type`; it never hosts or stores image binaries. | **CONFIRMED** | The domain owns the reference, not the bytes. |
| INV-IMG-03 | An `ItemImage` record depicts exactly one item. | **CONFIRMED** | It is a link table between item and image reference. |
| INV-IMG-04 | `Category` and `ItemType` each carry an `icon_url` and an `image_url`, so every application presents the vocabulary identically. | **CONFIRMED** | Two distinct fields: icon for UI chrome, image for headers/cards. |
| INV-IMG-05 | Images are added and removed only through Item Domain commands. | **PROPOSED** | Corollary of INV-INT-01. |
| INV-IMG-06 | Workflow state about images ("awaiting photography", "needs retouching", "photo approved") is never canonical. | **PROPOSED** | INV-ITEM-02 applied to images: the photo is canonical, producing it is not. |
| INV-IMG-07 | Ordering / primary designation among an item's images. | **OPEN** | `OQ-IMG-01`. Not modelled for now; forced by the first single-image view. |
| INV-IMG-08 | Whether the domain guarantees a URL resolves, and who repairs a dead reference. | **OPEN** | `OQ-IMG-04`. |
| INV-IMG-09 | Whether the same URL may be attached to multiple items; minimum/maximum image counts. | **OPEN** | `OQ-IMG-06`. |

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
| INV-INV-09 | Quantity is an ordinary count and is Inventory's **sole** responsibility; no other domain constrains it. One identity may cover many units. | **CONFIRMED** | `OQ-INV-01`. Mirrors INV-ITEM-14 from the Inventory side. |
| INV-INV-12 | ~~Quantity is presence: 0 or 1 per item, enforced as a bound.~~ | **WITHDRAWN** | Recorded briefly while "one Item = one unique physical piece" was read as capping quantity. Superseded by INV-ITEM-14: an Item identity may cover many units, and Item imposes nothing on quantity. |
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
| INV-INT-05 | Applications reference canonical entities by `item_id`, not by an external identifier or a business identifier used as a foreign key. | **CONFIRMED** (`item_id` stays the cross-system reference) / **PROPOSED** (migration detail) | During migration, legacy IDs may coexist but must be bridged via `ItemIdentifier`. `article_number` / `sku` may be *projected* for humans, but are not the key. |
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
| Item identity vs Inventory multiplicity | INV-ITEM-14 (an identity may cover one or many units) and INV-INV-09 (quantity is Inventory's alone) are the same boundary stated from both sides. Item answers *what canonical identity are these units represented as*; only Inventory answers *how many exist and where*. |
| No item relationships vs marriage/divorce | INV-ITEM-15 forbids item-to-item links *in this implementation*; the marriage/divorce machinery that would group or split identity lives outside this domain's scope (`OQ-ITEM-07`). This is also why INV-ITEM-12's "no merge" is not contradicted by marriage existing elsewhere. |
| Item vs Inventory ownership | INV-ITEM-11 and INV-INV-11 are complementary: Item has no quantity/location; Inventory has no description. No overlap. INV-ITEM-12 extends it: duplicates are never fused at the item level, because "how many" is Inventory's question. |
| Item deletion vs the forbidden Item → Inventory direction | INV-ITEM-13 keeps the boundary intact by *not* checking stock on delete. The cost — positions outliving a deleted item — is accepted deliberately, not overlooked (`OQ-ITEM-10`). |
| Item vs application workflow | INV-ITEM-02 (Item side) and INV-INT-02 (application side) together forbid workflow state in the domain and canonical authority in the app. |
| Canonical images vs photography workflow | INV-IMG-01 and INV-IMG-06 draw the line inside the same subject area: the resulting photo is canonical, the process that produces and approves it is not. INV-IMG-02 adds that the domain holds references, not files. |
| Location vs Party | INV-LOC-04 / INV-PARTY-02 are the same rule stated from both sides. Location → Party reference is PROPOSED, never the reverse (INV-PARTY-05). |
| Canonical ID vs external identifiers | Three layers, separated by issuing authority. INV-CID-01/02/03: our own `article_number` and `sku` are canonical attributes the domain guarantees. INV-ID-01/09: foreign-issued values are pointers, never identity, and never stored as canonical attributes. INV-INT-05: applications key off `item_id`, not off any other identifier. |
| Canonical state vs projections | INV-INT-02 + INV-INT-03 + INV-CC-07: projections are caches, applied idempotently by version. |
| Commands vs direct writes | INV-INT-01 + INV-ITEM-03 + INV-CLS-07: every write, including reference data, goes through commands. |
| Confirmed vs assumed | Every CONFIRMED row above traces to an explicit statement in the brief. Everything derived is PROPOSED; everything undecided is OPEN. |
