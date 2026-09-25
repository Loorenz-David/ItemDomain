# 05 — Location Domain

> **Question this domain answers:** *"What location is being referenced?"*
> **Status:** Minimal. **Location is the standard label that says who holds stock, what kind of place it is, and where it is**, for owned places and outside holders alike (`OQ-LOC-03`). Hierarchy and the type list are open.

## Purpose

The Inventory Domain and every application need to talk about locations — a warehouse, a store, a floor, a storage area, a dealer's or customer's premises. Without a shared authority, each application (and Inventory) would invent its own location identity, and "warehouse 2, floor 1, area B" would exist five times with five different IDs.

The Location Domain's **immediate architectural purpose** is narrow:

> Prevent every application and the Inventory Domain from independently defining location identity.

That is the whole job for now. It is intentionally **not** a warehouse-management system, a mapping system or an address book.

**Owned places and outside holders, in one table.** A location is either a place the company owns (a warehouse, a storage area) or an **outside holder** of stock — a customer, a dealer, a supplier. There is no separate Party domain for now (`OQ-LOC-03`, option A2). Every location answers three questions in the same way:

| Question | Field |
|---|---|
| **Who** holds it | `holder_type` (customer, dealer, supplier, …) plus name/address snapshots; the holder's record in its own system is linked through an **external identifier** of the location (e.g. `seller_app/customer_id = c_881`) — a polymorphic link. Empty for the company's own places. |
| **What kind** of place it is | `type` |
| **Where** it is | `name` (address and similar details open) |

The link is polymorphic because the kinds of holder will keep growing and splitting: this is a second-hand business, and anyone — a person on Facebook — can become a dealer. A new kind of holder is a new `holder_type`, not a schema change.

Inventory keeps holding the **quantity**; the location is only the label saying who, what kind and where.

## Owns

- Canonical `location_id` for every location referenced by the centralized domains.
- The minimal descriptive attributes needed to tell locations apart (a name; probably a type — open).
- Its own persistence.

## Does Not Own

| Not owned | Belongs to |
|---|---|
| What is *at* a location | [Inventory Domain](04-inventory-domain.md) |
| Which items exist | [Item Domain](01-item-domain.md) |
| Who a customer/dealer/supplier *is* (their full record) | The record the link points at, in the system that owns it. The location stores only `holder_type`, snapshots, and the record's id as an external identifier (`OQ-LOC-07`). |
| Routing, capacity, slotting, bin optimisation | Not in scope. |
| Application-specific location usage (e.g. "photography station 3" as a workflow stage) | Applications — unless that station is also a real place where stock sits, in which case it is a location *and* the application maps its stage to it. |

## Conceptual Model

Minimal, deliberately:

```
Location
    id                 ← canonical location_id
    name
    type               ← what kind of place: e.g. warehouse | storage_area | … | customer | dealer | supplier   (list open, OQ-LOC-02)
    parent_location_id ← optional; hierarchy is PROPOSED, not confirmed (OQ-LOC-01)
    holder_type        ← optional; which kind of holder — customer | dealer | supplier | …
                         (the holder's id in its own system is a LocationIdentifier, below)
    holder_name_snapshot    ← optional; the holder's name as given at registration
    address_snapshot        ← optional; the address as given at registration
                              (holder fields all empty for the company's own places — OQ-LOC-03)
    version

LocationIdentifier     ← external identifiers for locations, same pattern as items (OQ-LOC-06)
    id
    location_id
    type_id            ← → LocationIdentifierType (namespace + identifier_type), governed, users can add types
    value              ← stored exactly as the external system issued it
    source_id          ← → Source: the registering app, from its API key's request context (OQ-ID-05)
                         one value belongs to at most one location (OQ-LOC-06)
```

**The holder's record stays where it is.** The Location domain does not own customer, dealer or supplier records. It stores the kind of holder (`holder_type`), the holder's id in its own system **as an external identifier** (so one customer known as `c_881` in the Seller app and `7730` in Shopify is simply two identifiers on one location), and, **when the registration provides them**, a snapshot of the holder's name and address — the same approach as `user_name_snapshot` for users (`OQ-AUTHZ-02`). Location therefore knows who holds the stock without depending on another system's tables.

**External identifiers let apps share one location.** An app that creates a location registers its own id for it as an external identifier. Another app can find that existing location and register **its** own id on it, instead of creating a duplicate. A shelf barcode the Scanner reads is another external identifier of the same kind. **One external value belongs to at most one location**, so looking a location up by an app's id always gives one answer.

**Find before create.** Before creating a location, an app:

1. looks it up by **its own** external id → found: use it;
2. otherwise searches by name / address, to see whether another app already registered it → found: register its own id on it;
3. only then creates a new location, with its own id attached.

This is what keeps two apps from each creating their own "Dealer X" (`OQ-LOC-05`).

Potential examples from the brief: warehouse, store, floor, storage area, dealer/customer location, other operational location.

Whether these are **types** of one `Location` concept or **levels** of a hierarchy is exactly the kind of thing we are not deciding yet. The proposed direction is: one `Location` concept, a `type`, and an optional parent — which keeps both options open.

## Invariants

**Confirmed**

- `INV-LOC-01` — Location identity is defined only by the Location Domain; no application or other domain mints `location_id`s.
- `INV-LOC-07` — Every holder of stock — owned place, customer, dealer, supplier — is a location. Outside holders carry a `holder_type` and are linked to their record through an external identifier; Inventory only ever references `location_id`.
- `INV-LOC-08` — The holder's record stays in its own system; the location stores `holder_type`, the record's id as an external identifier, and name/address snapshots when provided.
- `INV-LOC-09` — Locations carry external identifiers, same pattern as items; one value belongs to at most one location.

**Proposed**

- `INV-LOC-02` — `location_id` is stable and immutable.
- `INV-LOC-03` — If a hierarchy is adopted, it is acyclic.

**Open**

- Whether stock may sit at every location type or only at "leaf"/storage-capable types (`OQ-LOC-04`).

## Commands / Operations

| Command (conceptual) | Notes |
|---|---|
| `CreateLocation { name, type, parent?, holder_type?, holder_name_snapshot?, address_snapshot?, identifiers[] }` | Only after *find before create*. A new customer, dealer or supplier that holds stock gets a location; the creating app attaches its own id. |
| `RenameLocation`, `MoveLocation (change parent)` | If hierarchy adopted. |
| `AddLocationIdentifier` / `RemoveLocationIdentifier` | An app attaches its own id to an existing location. Same pattern as items (`OQ-LOC-06`). |
| `RetireLocation` | What happens to inventory positions at a retired location is open. Note that **items** have no retire operation at all (`OQ-ITEM-04` resolved) — whether Location should differ, or follow the same soft-delete-only rule, is a consistency decision for the design session. |

## Queries

| Query | Who needs it |
|---|---|
| Get location by id | Inventory, all applications |
| List locations (optionally by type / parent) | Worker, Scanner (pick lists, labels), Manager |
| Resolve an external identifier → location | Any app looking for a location it (or another app) already registered; Scanner resolving a shelf barcode (`OQ-LOC-06`) |

## Events

**PROPOSED:** `LocationCreated`, `LocationChanged`, `LocationRetired` — so Inventory and applications can keep a local list of valid locations.

## Relationships

| Counterpart | Relationship |
|---|---|
| **Inventory** | Inventory → Location by `location_id`. This is the primary consumer. Inventory needs to know whether a `location_id` exists (and possibly whether it can hold stock). |
| **Holder records** | Location → holder (polymorphic, optional): an outside location points at the customer, dealer or supplier record behind it. Never the reverse. Where those records live is `OQ-LOC-07`. |
| **Item** | None. |
| **Applications** | Reference `location_id`; may hold projections of the location list. Scanner may need location identifiers/labels. |

## Example Flow

1. Manager creates `Location { name: "Main warehouse", type: warehouse }` → `loc_main`.
2. Manager creates `Location { name: "Storage area B2", type: storage_area, parent: loc_main }` → `loc_B2`.
3. Worker app, Scanner app and Inventory all refer to `loc_B2`. Nobody has their own "B2" record with a different ID.
4. A dealer takes stock on consignment: `Location { name: "Dealer X", type: dealer, holder_type: dealer, identifiers: [dealer_app/dealer_id = d_12] }` → `loc_dealer_x`. Inventory `transfer`s stock there.
5. A customer buys a piece: a `Location { type: customer, holder_type: customer, identifiers: [seller_app/customer_id = c_881] }` exists or is created, and Inventory moves the piece there. "Do we have it?" → no; "where is it?" → at that customer.

## Failure / Conflict Cases

| Case | Behaviour |
|---|---|
| Inventory movement to an unknown `location_id` | Rejected by Inventory (`INV-INV-06`); how it knows is `OQ-INV-06`. |
| Retiring a location that holds stock | Open. |
| Cyclic parent assignment | Rejected (`INV-LOC-03`, proposed). |
| Two applications create "the same" location twice | The Location Domain cannot detect this semantically without rules; governance and naming conventions are `OQ-LOC-05`. |

## Established Decisions

- Location is a separate, minimal domain providing canonical location identity.
- Inventory and applications reference locations by `location_id`.
- Location is **not** to be over-designed now.
- Location is the standard **who / what kind / where** label for every holder of stock — owned places and outside holders (customers, dealers, suppliers) alike. Outside holders are linked polymorphically. There is no Party domain for now.

## Open Questions

See [12-open-questions.md — Location](12-open-questions.md#location).

- `OQ-LOC-01` — Hierarchy: yes/no, and how deep.
- `OQ-LOC-02` — Location types.
- `OQ-LOC-03` — **Resolved:** no Party domain for now; outside holders are locations with a `holder_type`, linked to their record by an external identifier.
- `OQ-LOC-08` — **Resolved (mechanism):** each location type maps to *ours, on premises* / *ours, held by someone else* / *no longer ours*; "do we have it?" sums the first two. Relies on Inventory recording only owned or sold stock.
- `OQ-LOC-07` — **Resolved:** records stay in the system that owns them; the location stores `holder_type`, the record's id as an external identifier, and name/address snapshots when provided.
- `OQ-LOC-08` — Which location types count as "we have it".
- `OQ-LOC-04` — Virtual/logical locations (in transit, sold, lost) vs. movement sinks; which types can hold stock.
- `OQ-LOC-05` — Governance: who creates locations, via which application.
- `OQ-LOC-06` — **Resolved:** yes, same pattern as items; apps share one location by each registering their own id on it; one value → one location; find before create.
