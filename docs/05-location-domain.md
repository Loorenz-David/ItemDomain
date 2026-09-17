# 05 — Location Domain

> **Question this domain answers:** *"What location is being referenced?"*
> **Status:** Deliberately minimal. Identity only. Hierarchy, types and relationship to Party are **open**.

## Purpose

The Inventory Domain and every application need to talk about locations — a warehouse, a store, a floor, a storage area, a dealer's or customer's premises. Without a shared authority, each application (and Inventory) would invent its own location identity, and "warehouse 2, floor 1, area B" would exist five times with five different IDs.

The Location Domain's **immediate architectural purpose** is narrow:

> Prevent every application and the Inventory Domain from independently defining location identity.

That is the whole job for now. It is intentionally **not** a warehouse-management system, a mapping system or an address book.

## Owns

- Canonical `location_id` for every location referenced by the centralized domains.
- The minimal descriptive attributes needed to tell locations apart (a name; probably a type — open).
- Its own persistence.

## Does Not Own

| Not owned | Belongs to |
|---|---|
| What is *at* a location | [Inventory Domain](04-inventory-domain.md) |
| Which items exist | [Item Domain](01-item-domain.md) |
| Who a customer/dealer *is* | [Party Domain](06-party-domain.md). A "customer location" is a *location*; the customer is a *party*. The location may reference the party (proposed); it never defines it. |
| Routing, capacity, slotting, bin optimisation | Not in scope. |
| Application-specific location usage (e.g. "photography station 3" as a workflow stage) | Applications — unless that station is also a real place where stock sits, in which case it is a location *and* the application maps its stage to it. |

## Conceptual Model

Minimal, deliberately:

```
Location
    id                 ← canonical location_id
    name
    type               ← e.g. warehouse | store | floor | storage_area | customer_site | …   (list open, OQ-LOC-02)
    parent_location_id ← optional; hierarchy is PROPOSED, not confirmed (OQ-LOC-01)
    party_id           ← optional; for dealer/customer locations (PROPOSED, OQ-LOC-03)
    version
```

Potential examples from the brief: warehouse, store, floor, storage area, dealer/customer location, other operational location.

Whether these are **types** of one `Location` concept or **levels** of a hierarchy is exactly the kind of thing we are not deciding yet. The proposed direction is: one `Location` concept, a `type`, and an optional parent — which keeps both options open.

## Invariants

**Confirmed**

- `INV-LOC-01` — Location identity is defined only by the Location Domain; no application or other domain mints `location_id`s.
- `INV-LOC-04` — A location is never a party, and a party is never a location (`INV-PARTY-02` mirrors this). Follows from the two being separate bounded responsibilities.

**Proposed**

- `INV-LOC-02` — `location_id` is stable and immutable.
- `INV-LOC-03` — If a hierarchy is adopted, it is acyclic.

**Open**

- Whether stock may sit at every location type or only at "leaf"/storage-capable types (`OQ-LOC-04`).

## Commands / Operations

| Command (conceptual) | Notes |
|---|---|
| `CreateLocation { name, type, parent?, party? }` | Governance open (`OQ-LOC-05`). |
| `RenameLocation`, `MoveLocation (change parent)` | If hierarchy adopted. |
| `RetireLocation` | What happens to inventory positions at a retired location is open. |

## Queries

| Query | Who needs it |
|---|---|
| Get location by id | Inventory, all applications |
| List locations (optionally by type / parent) | Worker, Scanner (pick lists, labels), Manager |
| Resolve a scanned location code → `location_id` | Scanner — implies locations may need *identifiers* too (open; the item identifier pattern could be reused) |

## Events

**PROPOSED:** `LocationCreated`, `LocationChanged`, `LocationRetired` — so Inventory and applications can keep a local list of valid locations.

## Relationships

| Counterpart | Relationship |
|---|---|
| **Inventory** | Inventory → Location by `location_id`. This is the primary consumer. Inventory needs to know whether a `location_id` exists (and possibly whether it can hold stock). |
| **Party** | Location → Party (proposed, optional): a dealer/customer location may reference the party that owns/operates it. Direction is Location → Party, never the reverse. |
| **Item** | None. |
| **Applications** | Reference `location_id`; may hold projections of the location list. Scanner may need location identifiers/labels. |

## Example Flow

1. Manager creates `Location { name: "Main warehouse", type: warehouse }` → `loc_main`.
2. Manager creates `Location { name: "Storage area B2", type: storage_area, parent: loc_main }` → `loc_B2`.
3. Worker app, Scanner app and Inventory all refer to `loc_B2`. Nobody has their own "B2" record with a different ID.
4. A dealer takes stock on consignment (if that ever becomes a use case): `Location { name: "Dealer X showroom", type: customer_site, party: pty_dealer_x }`. Inventory can then `transfer` stock there; Party still knows who Dealer X is.

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
- Location is distinct from Party.

## Open Questions

See [12-open-questions.md — Location](12-open-questions.md#location).

- `OQ-LOC-01` — Hierarchy: yes/no, and how deep.
- `OQ-LOC-02` — Location types.
- `OQ-LOC-03` — Relationship to Party for dealer/customer locations.
- `OQ-LOC-04` — Virtual/logical locations (in transit, sold, lost) vs. movement sinks; which types can hold stock.
- `OQ-LOC-05` — Governance: who creates locations, via which application.
- `OQ-LOC-06` — Do locations need external identifiers (barcodes, legacy IDs) like items do?
