# 06 — Party Domain

> **⚠ Not built for now (`OQ-LOC-03`, option A2).** Outside holders of stock — customers, dealers, suppliers — are represented as **locations** with a polymorphic link to their record ([05](05-location-domain.md)). This document is kept as the record of the concept, in case a shared identity for people and organisations is needed later. Nothing else in the package depends on it.

> **Question this domain answers:** *"What person / organization / business entity is being referenced?"*
> **Status:** Deliberately immature. Identity only. **Not a CRM.** **Needed in phase 1, for Inventory** (`OQ-PARTY-01`): stock held by dealers or suppliers, and items sold to customers.

## Purpose

When the centralized domains need to refer to a person or organization — a supplier an item came from, a customer an item was sold to, a dealer holding stock — they need a canonical identity to refer to, for the same reason Item and Location exist: otherwise every application invents its own supplier/customer records and nothing can be joined.

The Party Domain provides that identity **and nothing more** until real requirements appear.

**Why it is needed now (`OQ-PARTY-01`).** Inventory will answer: *do we have this item? If yes, where? If not, is it with a dealer or a supplier (arrange delivery), or at a customer (sold)?* **Locations** are places the company owns; **parties** are entities outside that ownership that the company coordinates with, or whose holdings it has to understand. How Inventory records stock held by a party is open (`OQ-INV-08`, `OQ-LOC-03`). The Item Domain holds no party reference.

This document is deliberately short. Its main value is stating the boundary and listing what is not yet known.

## Owns

- Canonical `party_id` for every person/organization referenced by the centralized domains.
- The minimal attributes needed to distinguish parties (a display name; a kind — open).
- Its own persistence.

## Does Not Own

| Not owned | Belongs to / status |
|---|---|
| Customer relationship history, communication, notes, sales pipeline | Not in scope. **This is not a CRM.** |
| Orders, invoices, payments | Not assigned to a centralized domain in this architecture. |
| Locations belonging to a party | [Location Domain](05-location-domain.md) — a location may *reference* a party. |
| Inventory held by / on behalf of a party | [Inventory Domain](04-inventory-domain.md) — open whether movements reference parties (`OQ-INV-08`). |
| Items supplied by a party | Not recorded. The Item Domain holds no party reference (`OQ-PARTY-01`). |
| Application users / login identities / roles | Authorization concern (`OQ-AUTHZ-02`). A user who operates an application is not necessarily a Party. Do not conflate. |
| Personal data handling policies | Must be decided before any person data is stored (`OQ-PARTY-04`). |

## Conceptual Model

Minimal:

```
Party
    id                 ← canonical party_id
    kind               ← person | organization   (PROPOSED; or roles such as supplier/customer/dealer — OQ-PARTY-02)
    name
    version
```

Potential examples from the brief: supplier, customer, dealer, organization, person.

Note that "supplier", "customer" and "dealer" are **roles** a party plays toward the business, while "person" and "organization" are **kinds** of party. One organization can be both a supplier and a dealer. Whether roles are modelled at all, and whether they live on the party or on the referencing relationship, is `OQ-PARTY-02`. Do not decide this implicitly by choosing an enum.

## Invariants

**Confirmed**

- `INV-PARTY-01` — Party identity is defined only by the Party Domain.
- `INV-PARTY-02` — A party is not a location, and a location is not a party.

**Proposed**

- `INV-PARTY-03` — `party_id` is stable and immutable.

**Open**

- Essentially everything else.

## Commands / Operations

`CreateParty`, `RenameParty`, possibly `MergeParties` (duplicates are the norm in party data). All conceptual; none specified.

## Queries

- Get party by id.
- Search by name (for humans picking a supplier/customer).
- Resolve an external identifier (Shopify customer ID, supplier number) → `party_id` — **open** whether the item-identifier pattern is reused (`OQ-PARTY-03`).

## Events

**PROPOSED:** `PartyCreated`, `PartyChanged`, `PartyMerged`. Only needed once something consumes them.

## Relationships

| Counterpart | Relationship |
|---|---|
| **Location** | Location → Party (optional reference from a dealer/customer location). |
| **Inventory** | Possibly Inventory → Party on `sell`/`return`/`receive` movements. Open. |
| **Item** | Possibly Item → Party for supplier. Open. |
| **Applications** | Seller and Manager are the most likely to reference customers/dealers/suppliers. |

Party never references Item, Inventory or Location.

## Example Flow (illustrative only)

1. Manager records a new supplier: `CreateParty { kind: organization, name: "Nordic Furniture AB" }` → `pty_771`.
2. Inventory can then record stock held by that supplier, or an item sold to a customer, against the party (mechanism open, `OQ-INV-08`). The Item Domain carries no supplier reference (`OQ-PARTY-01`).
3. If a dealer location is modelled, `Location { type: customer_site, party_id: pty_771 }` links the place to the organization.

## Failure / Conflict Cases

- Duplicate parties (same organization entered twice) — expected; needs a strategy eventually. Open. Note that **items** have no merge operation (`OQ-ITEM-04` resolved); whether Party should differ, or also forbid merging, is a consistency decision for the design session.
- Referencing an unknown `party_id` from Location/Inventory — rejected; validation mechanism mirrors `OQ-INV-06`.

## Established Decisions

- Party is a separate, minimal domain providing canonical identity for people/organizations.
- It is **not** a CRM and must not grow into one without an explicit decision.
- Party is distinct from Location.

## Open Questions

See [12-open-questions.md — Party](12-open-questions.md#party).

- `OQ-PARTY-01` — **Resolved:** needed in phase 1 for Inventory (dealers, suppliers, customers); not referenced by Item.
- `OQ-PARTY-02` — Kinds vs roles; can a party hold several roles?
- `OQ-PARTY-03` — External identifiers for parties (e.g. Shopify customer ID).
- `OQ-PARTY-04` — Personal data / privacy requirements before any person data is stored.
- `OQ-PARTY-05` — Relationship between application users and parties.
