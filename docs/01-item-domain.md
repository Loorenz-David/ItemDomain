# 01 — Item Domain

> **Question this domain answers:** *"What is this thing?"*
> **Status:** Conceptual model established; persistence schema and API not finalized.

## Purpose

The Item Domain is the **single authoritative source for canonical item identity and canonical item description**.

It exists because every application currently holds its own representation of "an item", and there is no single record all of them can trust. The Item Domain gives every item one stable canonical identity (`item_id`), holds the facts that are *universally true* about the item regardless of which application is looking at it, and enforces the rules that keep those facts valid (classification rules, property rules, identifier rules, versioning).

Everything that is only true *inside one application's workflow* is explicitly excluded.

## Owns

The Item Domain is authoritative for:

| Concern | Detail |
|---|---|
| **Canonical surrogate identity** | The `item_id`. Stable, unique, never reused, minted by the domain. The identity applications store and reference. See [02](02-canonical-identity-and-identifiers.md). |
| **Canonical business identity** | `article_number` and `sku` — our own numbering schemes. First-class attributes of the Item, each independently unique. Values are strings, supplied by the creating application or person; the domain normalizes them and enforces uniqueness. They can be corrected; former values are kept. See [02](02-canonical-identity-and-identifiers.md). |
| **External identifiers attached to an item** | Shopify product/variant IDs, legacy system IDs, other foreign IDs — **any identity other than our `article_number` and `sku`**. Held as `ItemIdentifier` records that *point at* an item. Not identity. See [02](02-canonical-identity-and-identifiers.md). |
| **Classification** | Which `Category` and `ItemType` the item belongs to. See [03](03-classification-and-properties.md). |
| **Physical description** | `dimensions` (height, width, depth) and `weight` are conceptually first-class attributes in the initial model. |
| **Visual representation** | The images that show what the item is, held as ordered `ItemImage` link records: an absolute `url`, a `position`, and where it came from, where it is stored and how it is accessed. Canonical — an image is a fact about what the thing looks like. The domain owns the *references*, not the image files. See *Images* below. |
| **Classified properties** | Values for `PropertyDefinition`s that are legal for the item's classification (material, seat count, bulb type, …). See [03](03-classification-and-properties.md). |
| **Classification reference data** | `Category`, `ItemType`, `PropertyDefinition` — the vocabulary that determines what properties are legal. *Any application or user may change it; every change is recorded in the polymorphic history table (`OQ-CLS-02`).* |
| **Version and audit timestamps** | `version`, `created_at`, `updated_at` on the canonical entity. |
| **Its own persistence** | The Item database. No other component writes to it. |

## Does Not Own

This section is as important as the previous one.

| Not owned | Belongs to | Why |
|---|---|---|
| Wanted pieces the business doesn't own (wish list) | A separate wish-list application, later (`OQ-ITEM-13`) | A piece becomes an Item when acquired; `article_number` means "we have it". |
| Workflow state (`awaiting_photography`, `in_review`, `ready_for_listing`, …) | The application that runs the workflow (Worker, Manager, …) | Not a property of the item; a property of a process acting on the item. |
| Listing state (`draft`, `published`, `archived_on_shopify`) | Seller app / integration workers | Describes a sales channel process, not the item. |
| Task / assignment / queue state | Applications | Process state. |
| UI state, selections, filters, "selected for campaign" | Applications | Process/UI state. |
| **Quantity on hand, stock levels** | [Inventory Domain](04-inventory-domain.md) | "How much and where" is a different question from "what". |
| **Where the item physically is** | [Inventory Domain](04-inventory-domain.md) via `location_id` | Same reason. The Item Domain has no `location` field. |
| Location definitions | [Location Domain](05-location-domain.md) | |
| Supplier / customer / dealer identity | [Party Domain](06-party-domain.md) | The Item Domain may *reference* a party (open, `OQ-PARTY-01`) but never defines one. |
| Prices, sales history, orders | Not assigned to any centralized domain in this architecture | Out of scope for the Item Domain; do not add here without a boundary decision. |
| The image **files** themselves (hosting, storage, CDN, retention, deletion) | The application or third party that uploaded them | The Item Domain stores absolute URLs plus storage information; it never holds binaries. See `OQ-IMG-04`. |
| Workflow state about images ("awaiting photography", "needs retouching", "photo approved for listing") | Applications | The *photo* is canonical; the *process that produces and approves it* is not (`INV-IMG-06`). |
| Application-local projections of item data | Applications | Caches, not authorities. |

A useful test, applied to every candidate field:

> *"If every application were deleted tomorrow, would this fact still be true about the item?"*
> Dimensions: yes. Category: yes. `awaiting_photography`: no.

## Conceptual Model

This is a **conceptual** model. It is not a finalized schema; it names the concepts and their relationships.

```mermaid
classDiagram
    class Item {
        item_id
        article_number
        sku
        item_type_id
        dimensions : Dimensions
        weight_g
        version
        created_at
        updated_at
        deleted_at
    }
    class Dimensions {
        height_mm
        width_mm
        depth_mm
    }
    class ItemIdentifier {
        id
        item_id
        type_id
        value
        source_id
    }
    class ExternalIdentifierType {
        id
        namespace
        identifier_type
    }
    class ItemProperty {
        item_id
        property_definition_id
        value
    }
    class ItemBusinessIdentifierHistory {
        item_id
        kind
        value
        replaced_at
    }
    class ItemImage {
        id
        item_id
        url
        position
        state
        uploaded_by
        storage_type
        storage_key
        access_type
    }
    class Category {
        id
        name
        icon_url
    }
    class ItemType {
        id
        category_id
        name
        icon_url
    }
    class PropertyDefinition {
        id
        category_id / item_type_id
        name
        data_type
        unit
        required
        constraints
    }

    Item "1" --> "0..*" ItemIdentifier : external identifiers
    ItemIdentifier --> ExternalIdentifierType : of type
    Item "1" --> "0..*" ItemProperty : properties
    Item "1" --> "0..*" ItemImage : images
    Item "1" --> "0..*" ItemBusinessIdentifierHistory : former article_number / sku
    Item --> ItemType : classified as (category derived)
    ItemType --> Category : belongs to
    ItemProperty --> PropertyDefinition : conforms to
    PropertyDefinition --> Category : scoped to (and/or)
    PropertyDefinition --> ItemType : scoped to (and/or)
    Item *-- Dimensions
```

### Item

| Field | Meaning | Status |
|---|---|---|
| `item_id` | Canonical surrogate identity. Opaque, stable, never reused. Prefixed string, e.g. `itm_123`. | CONFIRMED. `itm_` + the record's UUIDv7 (`OQ-ID-07`). Examples shorten it to `itm_123` |
| `article_number` | Canonical business identity, assigned when the piece is **registered** at intake. Supplied, validated, unique. | CONFIRMED; required at creation. Non-empty string, no format rule, no max length; trimmed, lowercased, spaces → `-`; matched ignoring `-`; correctable, former values kept (`OQ-CID-03`/`04`/`05`) |
| `sku` | Canonical business identity, normally assigned when the restored piece is **published for sale**. | CONFIRMED; optional — may be sent at creation, and can be cleared (`INV-CID-07`, `INV-ITEM-25`). Same string, normalization and correction rules as `article_number` |
| `item_type_id` | Reference to `ItemType`. The item's **category is derived** from it (`item_type.category_id`) and is not stored on the item. | CONFIRMED (`OQ-CLS-05`) |
| `dimensions.height_mm / width_mm / depth_mm` | Physical dimensions in **millimetres**. **Universal** canonical attribute — every object has them, whatever type it is. | CONFIRMED. Each dimension optional on its own — an item may have only some. Whole number greater than zero when present; can be cleared (`INV-ITEM-24`, `INV-ITEM-25`). Units fixed (`INV-ITEM-23`) |
| `weight_g` | Physical weight in **grams**. Universal canonical attribute. | CONFIRMED. Optional; whole number greater than zero when present; can be cleared (`INV-ITEM-24`, `INV-ITEM-25`). Units fixed (`INV-ITEM-23`) |
| `identifiers[]` | The set of **external** `ItemIdentifier` attached to this item. | CONFIRMED concept |
| former business identifiers | Every previous `article_number` / `sku` value, kept when a value is corrected. Still matched by lookups and still unique. | CONFIRMED concept (`OQ-CID-04`) |
| `properties[]` | The set of `ItemProperty` attached to this item. | CONFIRMED concept |
| `images[]` | The set of `ItemImage` attached to this item. | CONFIRMED concept |
| `version` | Monotonic version for optimistic concurrency. Every change to the item or its identifiers, properties or images bumps it. | CONFIRMED (`INV-ITEM-07`) |
| `created_at`, `updated_at` | Audit timestamps. | PROPOSED |
| `deleted_at` | Soft-delete marker. Every deletion is a soft delete; the record is retained. A deleted item accepts no changes. | CONFIRMED (`INV-ITEM-10`, `INV-ITEM-26`) |

**An Item is a canonical business identity, not a count of physical units.**

> An Item is the canonical identity assigned by the business to **one or more** physical units that are currently treated as the same thing for identification purposes. The number of units represented by that identity is exclusively an Inventory concern.

So an Item may stand for one unique physical piece, or for sixty interchangeable ones. The business may purchase and register 60 identical chairs under **one** Item — one `item_id`, one `article_number`, eventually one `sku`. From inside the Item Domain those sixty chairs are simply *one canonical identity*: the domain does not represent the fact that there are sixty at all. That number lives in `InventoryPosition.quantity` and nowhere else.

| Domain | Question it answers |
|---|---|
| **Item** | *What canonical identity are these units represented as?* |
| **Inventory** | *How many units of that identity exist, and where are they?* |

**There is no invariant that one Item equals one physical unit** (`INV-ITEM-14`), and nothing in the model may assume one. Unique vintage furniture commonly produces one Item per physical piece — two visually similar vintage chairs will usually be registered as two items — but that is a **business usage pattern, not a domain rule**.

**The business decides, the domain records.** Identity grouping is a decision made when items are registered and managed. The Item Domain records and enforces the resulting identity; it never infers whether two physical units are similar enough to share one (`OQ-ITEM-12`).

**No relationships between Items.** An Item is an independent canonical row: no variants, no sets, no components, no parent/child items, no generic item-to-item links (`INV-ITEM-15`). Deciding to restore and sell four chairs as a set is a commercial act, not a persistent link between rows. Where item *identity* genuinely has to be grouped or split, that is separate business machinery — **marriage** and **divorce** — described below and **out of scope for this implementation** (`OQ-ITEM-07`).

**Splitting and grouping identity.** The business may later decide that units currently held under one identity should become separately identified. For example: Item A carries article number `CHAIR-001` and Inventory records quantity 60; later, four of those chairs are restored and sold as one separately identified group while the rest stay under the original identity. That can produce **new Item identities with their own `item_id`, `article_number` and potentially `sku`**. The inverse — grouping identities together — may exist too.

> **Do not design this yet.** How quantities move, how new identities are created, whether identifiers are inherited or regenerated, how history is represented, the transactional boundaries, and which domain or application owns the operation are all to be defined later. For this implementation, record only that the machinery exists conceptually, that it is out of scope, and that it **must not be pre-modelled** — no item-to-item relationships, parent/child links, sets or variants introduced in anticipation of it (`INV-ITEM-15`, `INV-ITEM-21`).

The Item model therefore carries **no product-definition or grouping semantics**: no "product with variants", no shared parent record.

The two business identifiers mark different points in that single piece's life:

| Identifier | Assigned when | Says |
|---|---|---|
| `article_number` | the piece is created / registered at intake | we have it and are tracking it |
| `sku` | the restored piece is published for sale | it is being offered |

This is the normal pattern, not a rule the domain enforces: `sku` is optional and may be sent at creation (e.g. when migrating a piece that is already on sale).

> **Caution — `sku` is not a workflow flag.** It is *assigned at* a workflow milestone, but its presence is not a status. The Item Domain knows nothing about publishing: it has no `published` or `listed` state and must never gain one (`INV-ITEM-02`). An application must not read `sku IS NULL` as "not yet listed" — listing state belongs to the Seller application. At most, a missing `sku` says the piece has never been published, which is a historical fact rather than current status.

**Deliberately absent: `catalogue_id`.** Grouping similar pieces — potentially by similarity or vectorization over the properties that define them — is a **future concept and explicitly not part of this implementation**. It is named here so nobody reinvents it under another name, and so nothing is built now that approximates grouping. Adding it later is a nullable reference on the item plus a new entity; nothing in the current model needs to anticipate it. See `OQ-ITEM-11`.

**Deliberately absent: `name` and `description`.** Neither exists in this implementation. Classification, properties and images already describe what the thing is, and free text adds nothing the domain can validate or enforce — it is precisely the kind of field that drifts between applications while meaning nothing in particular.

A human recognises a piece from **its type, its image, and its article number or SKU** (`OQ-ITEM-09`). That is the whole presentation contract: identity is `article_number` / `sku`, what it is is classification plus properties, what it looks like is images.

If the business later needs something name-like — a model or range name such as "Stockholm" — its natural home is a `PropertyDefinition` scoped to the item types that actually have one, validated like any other property. That keeps naming classification-driven instead of a universal free-text column. See `OQ-ITEM-02` (resolved) and `OQ-ITEM-09`.

**Universal vs conditional.** `dimensions` and `weight` are first-class attributes of the Item because every physical object has them, whatever kind of thing it is. Properties are **conditional** attributes, meaningful only given the item's category and type — a seat count says nothing about a lamp. That is the test for any future candidate field: universal physical fact → first-class attribute; type-conditional fact → property (`OQ-ITEM-03`).

**Units are in the field name.** Dimensions are always millimetres and weight always grams, domain-wide, and the field names say so (`height_mm`, `width_mm`, `depth_mm`, `weight_g`). The domain stores no unit alongside a value and does no conversion: a client that lets a person type "22 cm" or "4.5 kg" converts to mm and g before sending, and converts back for display. Protecting those values is the client's responsibility (`INV-ITEM-23`).

### Images

**ESTABLISHED:** an item's images are **canonical**. What the item looks like is a fact about the thing itself, not about any application's process, so it passes the same test as dimensions: if every application were deleted tomorrow, the sofa would still look like that.

```
ItemImage
    id
    item_id            ← the item this image depicts (link table)
    url                ← absolute URL; required from clients for now
    position           ← display order, gap-free (1, 2, 3 …); assigned from the list index when the client sends none (OQ-IMG-01)
    state              ← available | not_available — set by the URL check (OQ-IMG-04)
    uploaded_by        ← the source that uploaded it, from the API key's request context (OQ-ID-05)
    storage_type       ← the storage the file lives in
    storage_key        ← optional; lets the reference be rebuilt from storage_type + key (OQ-IMG-04)
    access_type        ← how the file can be accessed               e.g. public URL today; private later
```

- **Order is canonical.** Every application shows an item's images in `position` order, and the first is the one shown when only one fits. Users can reorder; reordering is an item change. The domain keeps positions gap-free: removing an image closes the gap, inserting at a taken position shifts the rest (`OQ-IMG-01`).
- **One URL, several items.** The same URL may be attached to more than one item, one `ItemImage` record per item. An item may have **no** images, and there is no maximum (`OQ-IMG-06`).
- **Checked on attach.** The URL is checked when the image is attached; if it does not resolve, the image is kept with `state = not_available`. The checker is standalone, ready for a later background job (`OQ-IMG-04`).
- **Ready for private storage.** Clients must send absolute URLs today, but `storage_type` + `storage_key` + `access_type` let a later private-storage setup rebuild or sign references without a schema change (`OQ-IMG-04`, `OQ-IMG-05`).

Two consequences worth stating explicitly:

- **The domain owns the reference, not the bytes.** Image files live in application-owned or third-party storage. The Item Domain records an absolute URL and never holds binaries. This keeps the domain simple, but it means a piece of canonical state depends on infrastructure the domain does not control: a retired bucket or a rotated CDN path silently breaks canonical data. `uploaded_by`, `storage_type` and `storage_key` make this tractable — a broken reference can be attributed and potentially rebuilt. Detecting a dead URL is still open (`OQ-IMG-04`).
- **The photo is canonical; producing it is not.** "Awaiting photography", "needs retouching" and "photo approved for listing" are workflow states belonging to the Worker and Seller applications. The resulting image is canonical; the process is not (`INV-IMG-06`).

Deliberately **not** modelled for now — role or purpose such as front/detail/damage (`OQ-IMG-02`), and alt text / captions (`OQ-IMG-03`).

### Aggregate boundary

**CONFIRMED:** `Item` is the aggregate root. `ItemIdentifier`, `ItemProperty` and `ItemImage` are owned by the item and are modified *through* the item, so that:

- a single command can be validated against the whole item state (e.g. "is this property legal for this item's type?");
- a single `version` can guard concurrent modification of the item and its identifiers/properties together.

**Identifier, property and image changes all increment `item.version`** — that is what the version is for. It tracks the item's canonical state as a whole, so a change to any part of it is a change to the item (`OQ-ITEM-05` resolved, `INV-ITEM-07`).

The contention this creates is accepted deliberately: two applications editing different parts of the same item *will* conflict, and the alternative is silent divergence. In practice a Worker uploading eight photographs produces eight version bumps, any of which can invalidate a concurrent edit elsewhere, so applications need a re-read-and-retry path rather than a one-shot write. It also raises the stakes on command granularity (`OQ-API-02`): a coarse `UpdateItem` saving a whole form is one bump where fine-grained commands are many.

`Category`, `ItemType` and `PropertyDefinition` are **reference data**, not part of the item aggregate. They are owned by the Item Domain but have their own lifecycle. See [03](03-classification-and-properties.md).

## Invariants

Full list with IDs in [11-invariants.md](11-invariants.md#item). Summary:

**Confirmed**

- `INV-ITEM-01` — A canonical `item_id` identifies exactly one item and is never reused.
- `INV-ITEM-02` — Application workflow state is never stored as canonical item state.
- `INV-ITEM-03` — Every change to canonical item state passes through the Item Domain's command boundary.
- `INV-ITEM-04` — Every item has a version; a committed change to the item increments it.
- `INV-CID-01` — `article_number` and `sku` are canonical identity and are first-class attributes of the Item.
- `INV-CID-02` — Each of `article_number` and `sku` identifies at most one item.
- `INV-CID-03` — The domain enforces uniqueness and normalization on supplied business identifiers; it does not generate them.
- `INV-CID-05` — `article_number` and `sku` can be corrected; former values are kept and still matched by lookups.
- `INV-CID-09` — Values are lowercased and spaces replaced with `-`, on write and on lookup.
- `INV-ID-01` — External identifiers are never the canonical identity.
- `INV-CLS-01` — Properties assigned to an item must conform to the property definitions allowed for the item's classification.
- `INV-ITEM-06` — An item's category is its item type's category (holds by construction: no `category_id` is stored).
- `INV-ITEM-07` — Identifier, property and image changes are changes to the item aggregate and increment its version.
- `INV-ITEM-09` — An item always has a classification.
- `INV-ITEM-10` / `INV-ITEM-12` — Every deletion is a soft delete; there is no retire and no merge.
- `INV-ITEM-16` — Mandatory at creation: `article_number`, `item_type`, required properties. Optional fields may also be sent.
- `INV-ITEM-23` — Dimensions in mm, weight in g, unit in the field name; clients convert.
- `INV-ITEM-24` — Each dimension and the weight is optional on its own; a whole number greater than zero when present.
- `INV-ITEM-25` — `sku`, dimensions and weight can be cleared.
- `INV-ITEM-26` — A soft-deleted item accepts no changes except restore.
- `INV-ITEM-05` — `item_id` is the item record's own id, generated by the domain; it never changes and carries no meaning.
- `INV-ITEM-08` — An item has exactly one item type at a time.

## Commands / Operations

Commands are **conceptual**. They are built as **atomic services** (one responsibility each), grouped by an **orchestrating `UpdateItem`** that applies several in one transaction with one version bump and one event (`OQ-API-02`, `OQ-EVT-01`). Each command is subject to authentication by API key — every registered app may issue every command for now (`OQ-AUTHZ-01`) — validation, optimistic concurrency ([09](09-versioning-concurrency-and-idempotency.md)) and emits events ([08](08-commands-queries-and-events.md)).

| Command (conceptual) | Effect | Notes |
|---|---|---|
| `CreateItem` | Registers one canonical identity: mints an `item_id` and carries the supplied `article_number`. | **Mandatory:** `article_number`, `item_type` (the category follows from it), and all **required properties** for that classification (`INV-ITEM-16`). **Optional** fields — `sku`, any dimensions, weight, identifiers, images, other properties — may also be sent. |
| `SetItemArticleNumber` / `SetItemSku` (or via `UpdateItem`) | Corrects a canonical business identifier (e.g. a typo), or clears `sku`. | The new value is normalized and must not be held — now or formerly — by another item. The old value is kept as a former value (`OQ-CID-04`). `article_number` cannot be cleared. |
| `ClassifyItem` / `ChangeClassification` | Changes `item_type_id`; the category follows from the new type. | Same-named valid values carry over; others are recorded in the history table and removed; missing required values send the item to the resolution list (`OQ-CLS-03`). Also happens in bulk when a type with items is deleted. |
| `UpdateItemDimensions` / `UpdateItemWeight` (or a general `UpdateItem`) | Sets, changes or clears any dimension or the weight. | Any subset of dimensions may be set; values must be > 0 (`INV-ITEM-24`). Coarse vs fine-grained commands is open (`OQ-API-02`). |
| `SetItemProperty` / `RemoveItemProperty` | Sets or clears a value for a `PropertyDefinition`. | Validated against the item type's definitions: data type, choices, one-or-several picks (`OQ-PROP-02`). |
| `AddItemIdentifier` | Attaches an **external** identifier. | The type must exist. The same value may already be on other items (`OQ-ID-01`). Rejects canonical identities (`INV-ID-09`). |
| `RemoveItemIdentifier` | Detaches an external identifier. | History retention open (`OQ-ID-06`). |
| `AddItemImage` | Attaches one or more images (absolute `url`, optional `position`, `storage_type`, optional `storage_key`, `access_type`). | Without a `position`, it is assigned from the list index (`OQ-IMG-01`). The URL may already be on other items (`OQ-IMG-06`). The URL is checked; a failure marks the image `not_available` (`OQ-IMG-04`). |
| `ReorderItemImages` | Changes the `position` of an item's images. | Increments `item.version` like any image change. |
| `RemoveItemImage` | Detaches an image. | Whether the underlying file is deleted from storage is the uploading application's concern, not the domain's. |
| Reference-data commands (`CreateCategory`, `CreateItemType`, `DefineProperty`, …) | Manage classification vocabulary. | Open to any application or user; recorded in the polymorphic history table (`OQ-CLS-02`). Definition changes may leave items non-conforming — see the resolution endpoints in [03](03-classification-and-properties.md) (`OQ-PROP-03`). |
| `UpdateItem` | Orchestrator: applies several atomic changes in one transaction. | One version bump; one `ItemUpdated` event naming the changed parts; the atomic commands' own events are not broadcast. |
| `RestoreItem` | Undeletes a soft-deleted item. | The only change a deleted item accepts (`INV-ITEM-26`). Emits `ItemRestored`. |
| `DeleteItem` | Soft-deletes the item; the record is retained and flagged. | **CONFIRMED.** Does not consult Inventory (`INV-ITEM-13`). Deleted items are excluded from queries by default; callers can opt in (`INV-ITEM-18`). After deletion the item accepts no changes (`INV-ITEM-26`). |
| ~~`RetireItem`~~, ~~`MergeItems`~~ | **These do not exist.** | Items are never retired or merged. If the business has more of a thing, that is a quantity fact in [Inventory](04-inventory-domain.md), not an operation on the item record. |

**Not accepted, by design:**

- Any command that sets workflow state on an item.
- Any command that sets quantity or location on an item (that is Inventory).

## Queries

Consumers need, at minimum:

| Query (conceptual) | Who needs it |
|---|---|
| Get item by `item_id` (full canonical state incl. identifiers, properties, version) | All applications |
| **Get item by `article_number` / by `sku`** — direct lookup on a canonical attribute. Input is normalized; matches current **and former** values and says which | Scanner, Manager, Seller |
| **Resolve external identifier → list of items** (`namespace`, `identifier_type`, `value`) — a value may be on several items | Integrations, every application during migration |
| List external identifiers for an item | Integrations, Seller |
| Get classification reference data (categories, item types, property definitions for a type) | Manager, Worker (forms/validation), Seller |
| Search / filter items — type, category, property value (by name across all types when no type is given), identifier value or prefix, deleted-or-not; paging and sorting. Free text later (`OQ-API-04`) | Manager, Seller |
| Batch get by `item_id[]` | Projections, bulk views |

Whether applications should query the API synchronously on hot paths or rely on their projections is open (`OQ-APP-02`).

## Events

One event per atomic command; the orchestrating `UpdateItem` silences those and broadcasts a single `ItemUpdated` naming the changed parts. **Every event carries the full item after the change** (`OQ-EVT-01`, `OQ-EVT-02`). Initial names:

| Event | Emitted when |
|---|---|
| `ItemCreated` | A new canonical item exists. |
| `ItemUpdated` | The orchestrator applied several changes; lists which. |
| `ItemIdentifierAdded` | An identifier was attached. |
| `ItemIdentifierRemoved` | An identifier was detached. |
| `ItemClassificationChanged` | The item type changed (and with it, possibly the derived category). |
| `ItemPropertyChanged` | A property value was set, changed or removed. |
| `ItemRestored` | A soft-deleted item was restored. |
| `ItemDeleted` | The item was soft-deleted. **Proposed** name; the fact itself is confirmed. Consumers must decide whether to drop or flag the projection row (`OQ-ITEM-10`). |
| `ItemImageAdded` / `ItemImageRemoved` | An image was attached or detached. **Proposed** — not in the brief's initial list, but image changes are canonical state changes and consumers (Seller, Shopify) will need them. |

Every event carries the `item_id`, the resulting `version`, the acting app and `user_name_snapshot`, and the full item, so consumers can apply them idempotently and in order. See [08](08-commands-queries-and-events.md).

## Relationships

| Counterpart | Direction | Nature |
|---|---|---|
| **Inventory Domain** | Inventory → Item | Inventory references `item_id`. Item knows nothing about inventory. |
| **Location Domain** | none | Item does not reference locations. |
| **Party Domain** | none | The item holds no party reference. Party is needed by Inventory — stock at dealers or suppliers, items sold to customers (`OQ-PARTY-01`). |
| **Applications** | App → Item (commands, queries); Item → App (events) | Applications hold `item_id` and projections. |
| **Shopify** | via integration workers | Shopify product/variant IDs are identifiers in the `shopify` namespace. Shopify both **supplies** and **consumes** item facts, like every connected app; the Item Domain stays the only authority (`OQ-MIG-04`). |

## Example Flow

### Flow A — A worker measures a sofa

1. The Worker app shows a task "measure item `itm_123`" (workflow state owned by Worker).
2. The worker enters 220 × 90 × 85 cm.
3. Worker app sends `UpdateItemDimensions { item_id: itm_123, expected_version: 17, dimensions: {…}, idempotency_key: … }` to the Item API.
   The Worker app converts to the domain's units before sending: `{ height_mm: 2200, width_mm: 900, depth_mm: 850 }`.
4. Item Domain: authorizes the Worker app, validates the payload, checks `current_version == 17`, applies the change, sets `version = 18`, writes `ItemUpdated { item_id, version: 18, … }` to the outbox — all in one transaction.
5. Item API responds with `version: 18`.
6. Worker app advances *its own* workflow state to `awaiting_photography` in *its own* database. The Item Domain never learns about this.
7. Outbox publishes `ItemUpdated`. Seller app's projection updates the cached dimensions for `itm_123` to version 18.

### Flow B — Scanner reads a label

1. Scanner reads a label carrying our article number `A-4932`.
2. Scanner app queries `GetItemByArticleNumber("A-4932")`.
3. Item Domain normalizes the input to `a-4932` and returns the item (`item_id: itm_123`), or "not found". It cannot be ambiguous: `article_number` is canonical identity and therefore unique, former values included (`INV-CID-02`, `INV-CID-06`). If the label carries a value that was later corrected, the item is still found and the response says it matched a former value.
4. Scanner app proceeds with its own workflow using `itm_123`; it may then issue an *Inventory* command such as `place()` — not an Item command.

Had the label carried a *foreign* code (a legacy system's sticker), step 2 would instead be `ResolveExternalIdentifier(namespace, identifier_type, value)`, which returns a **list** — the same external value may be on several items (`OQ-ID-01`).

### Flow C — Creating an item with an initial classification (illustrative)

1. Manager app sends `CreateItem { article_number: "A-4932", sku: "SOFA-STO-3L", item_type_id: sofa, properties: { material: "leather", seat_count: 3 }, identifiers: [{ namespace: "shopify", identifier_type: "variant_id", value: "482901284" }] }` — the identifier's source is the calling app, from its API key. `sku` and the identifier are optional; they are sent here because this piece is already listed.
2. Item Domain validates: `article_number` and `sku` are not already held by another item; item type exists (its category is derived); `material` and `seat_count` are defined for Sofa; values match data types/constraints; all required properties for Sofa are present (`INV-CLS-05`); the identifier's type exists in `ExternalIdentifierType` and its value does not violate uniqueness rules (open, `OQ-ID-01`).
3. Item is created with `version: 1`; `ItemCreated` (and possibly finer-grained events) written to outbox.

## Failure / Conflict Cases

| Case | Expected behaviour |
|---|---|
| Update with `expected_version` ≠ current version | Rejected as a **version conflict**. Caller must re-read and decide. See [09](09-versioning-concurrency-and-idempotency.md). |
| Property not defined for the item's classification (e.g. `bulb_type` on a Sofa) | Rejected as a **validation error**. |
| Property value violates data type / constraints | Rejected as a **validation error**. |
| `article_number` or `sku` already held by another item — currently or formerly, compared after normalization | Rejected as a **conflict**. Uniqueness is what makes them identity (`INV-CID-02`, `INV-CID-06`). |
| External identifier value already on another item | **Accepted** — values may be shared (`OQ-ID-01`). |
| Image URL does not resolve when attached | Image kept, marked `state = not_available` (`OQ-IMG-04`). |
| Unknown `item_type_id` | Rejected as a **validation error**. (Type/category disagreement cannot occur: the category is derived, `INV-ITEM-06`.) |
| `CreateItem` without a classification | Rejected — an item cannot exist unclassified (`INV-ITEM-09`). |
| Property with no definition on the item's type | Rejected — free-form properties do not exist (`OQ-PROP-04`). |
| Empty `article_number`, `sku` or external identifier value | Rejected as a **validation error** (`OQ-CID-03`). |
| `CreateItem` missing a required property for its classification | Rejected — required properties are enforced at creation (`INV-CLS-05`). |
| `DeleteItem` on an already-deleted item | Idempotent no-op (proposed). |
| Any other change to a soft-deleted item | Rejected (`INV-ITEM-26`). |
| A dimension or weight of zero or less | Rejected as a **validation error** (`INV-ITEM-24`). |
| `DeleteItem` while Inventory still holds stock | **Succeeds.** Item never reads Inventory state (`INV-ITEM-13`), so positions can outlive the item record. Reconciliation is a deliberate cost of the boundary (`OQ-ITEM-10`). |
| Attempt to set a workflow-like attribute (`status = awaiting_photography`) | There is no such command; the API must not offer a generic "set arbitrary attribute" escape hatch that would allow it. |
| Reclassification leaves property values the new type doesn't define | Recorded in the history table and removed; the move succeeds (`OQ-CLS-03`). |
| Retried command with the same idempotency key | Same result returned; no second change. See [09](09-versioning-concurrency-and-idempotency.md). |
| Unknown `item_id` | Not found. |

## Established Decisions

- The Item Domain answers *"what is this thing?"* and nothing else.
- It owns canonical item identity in three layers: the domain-minted surrogate `item_id`, the supplied-but-guaranteed business identities `article_number` and `sku`, and recorded external identifiers.
- `article_number` and `sku` are canonical identity — two distinct, independently unique, first-class attributes of the Item.
- External identifiers (Shopify IDs, legacy app IDs — any identity other than `article_number` and `sku`) are attached to items and are never the canonical identity.
- `item_id` remains the identity applications store and reference across systems.
- It does **not** own application workflow state, quantity, or location.
- It owns its persistence; all writes go through its API.
- Properties are classification-driven and definition-based; there is no giant item table with every possible column.
- **An Item is the canonical identity the business assigns to one or more physical units treated as the same thing for identification purposes.** There is **no invariant that one Item equals one physical unit**; multiple units may share one `item_id`, `article_number` and `sku`. One Item per unique vintage piece is a common usage pattern, not a domain rule.
- **Quantity never belongs to the Item Domain.** How many units exist under an identity is exclusively Inventory's concern.
- **Identity grouping is a business decision** made at registration/management. The domain records and enforces it; it never infers it.
- The business may later **split or group** identities via "marriage / divorce" machinery. Its semantics are undefined, out of scope, and must not be pre-modelled here.
- **Items have no relationships to other Items.** No variants, sets, components or parent/child links. Grouping or splitting identity is separate "marriage / divorce" machinery, out of scope here.
- The model carries no product-definition or grouping semantics; `catalogue_id` is a deferred future concept.
- `dimensions` and `weight` are **universal** first-class attributes; properties are **conditional** on category and type.
- Dimensions are in **millimetres** and weight in **grams**, domain-wide, with the unit in the field name (`height_mm`, `width_mm`, `depth_mm`, `weight_g`). Clients convert; the domain does not.
- `item_id` is `itm_` + a UUIDv7, minted by the domain.
- The item stores `item_type_id` only; its **category is derived** from the item type.
- Identifier, property and image changes **all increment `item.version`**.
- Mandatory at creation: `article_number`, `item_type`, and required properties — nothing else. Optional fields may be sent at creation too.
- Each dimension and the weight is optional on its own and must be a whole number greater than zero when present; `sku`, dimensions and weight can be cleared.
- A soft-deleted item accepts no changes except restore (`RestoreItem`).
- Soft deletion follows the industry standard: a deletion marker, excluded from queries by default, opt-in to include, restore possible.
- `article_number` is assigned at registration and is required at creation; `sku` is normally assigned when the restored piece is published for sale, and is optional.
- An item **cannot exist without a classification**; the item type is required at creation, and it determines the category.
- Items **can be deleted, and every deletion is a soft delete** — the record is retained and flagged, never physically removed.
- There is **no retire and no merge**. Two item records are never fused; "more of this thing" is a quantity fact in the Inventory Domain.
- Deleting an item does **not** consult Inventory; the Item Domain never reads Inventory state.
- `article_number` and `sku` are permanently unique: a soft-deleted item keeps its values and they are never reused.
- `article_number` and `sku` are strings with no format rule. They are normalized (lowercase, spaces → `-`) on write and on lookup, and can be corrected; former values are kept, stay unique, and are still found by lookups.
- Item images are canonical: an item's visual representation is part of its canonical description. They are held as `ItemImage` link records carrying an absolute `url`, a `position`, `uploaded_by`, `storage_type`, an optional `storage_key` and `access_type`. The same URL may be on several items; an item needs no images. The domain owns the references, not the image files.
- `Category` and `ItemType` each carry one optional `icon_url`; when missing, the API returns none.
- Canonical items carry a version and updates use optimistic concurrency.
- Canonical state changes are published as domain events via a transactional outbox.
- The conceptual model above is the starting point; the exact schema is *not* finalized.

## Open Questions

See [12-open-questions.md](12-open-questions.md) for full detail.

**Still open — affects the Item Domain**

Nothing that changes the Item Domain's shape. Small details that can be settled during the build:

- two identifiers of one type from one source on one item (`OQ-ID-02`);
- names and value lists of the image storage fields (`OQ-IMG-05`);
- how non-conforming items are detected, and whether other writes are allowed on them meanwhile (`OQ-PROP-03`);
- reference-data event names (`OQ-EVT-01`);
- recording direct users' ids as well as names (`OQ-AUTHZ-02`);
- whether one API key works across all domains (proposed: yes, `OQ-AUTHZ-01`).

**Deferred**

- `OQ-ITEM-10` residuals — projection behaviour on delete, and who reconciles inventory left behind.
- `OQ-ITEM-11` — `catalogue_id`, grouping similar pieces.
- `OQ-ID-06` — History of removed external identifiers.
- `OQ-IMG-02` / `03` — Image role, alt text. Background re-check of image URLs (`OQ-IMG-04`).

**Resolved** — `OQ-ITEM-01` … `10` (mechanism, incl. restore), `12`, `13` (wish list lives in its own app); `OQ-CID-01` … `05`; `OQ-ID-01` … `05`, `07`, `08` … `10`; `OQ-CLS-02`, `03`, `05`; `OQ-PROP-01` … `04`, `06`; `OQ-IMG-01`, `05`, `06`, `07`; `OQ-API-02`, `04`; `OQ-EVT-01` (items), `02`; `OQ-AUTHZ-01`, `02`; `OQ-PARTY-01`; `OQ-MIG-04`.
