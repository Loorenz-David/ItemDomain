# 02 — Canonical Identity and Identifiers

> **Responsibility:** Provide one stable canonical identity per item, and let every existing identifier in the business resolve to it.
> **Status:** Concept established. Uniqueness rules, namespace registry and identifier lifecycle are **open**.

## Purpose

Today an item is known by many names: an article number in one app, a SKU in another, a Shopify variant ID in the shop, a supplier's article number on the invoice, a legacy ID in an old system. None of these is a good canonical identity:

- they are assigned by different systems with different rules;
- they can change, be reused, or be missing;
- several may exist for the same thing;
- some are only meaningful inside one channel (Shopify).

The Item Domain therefore separates two concepts:

| Concept | Owner | Purpose |
|---|---|---|
| **Canonical identity** — `item_id` | Item Domain, assigned at creation | The one ID every system uses to refer to the item. |
| **Identifiers** — `ItemIdentifier` | Item Domain, attached to an item | Every other way the business refers to the item, recorded *as pointers to* the canonical identity. |

This separation is what allows existing applications to keep their historical identifiers and still converge on one canonical identity **without a big-bang migration**: an application resolves what it has (e.g. `shopify/variant_id/482901284`) into an `item_id`, and from then on stores the `item_id`.

## Owns

- Generation and guarantee of uniqueness of `item_id`.
- The set of `ItemIdentifier` records and the rules governing them (namespace/type semantics, uniqueness, lifecycle).
- Resolution of an identifier to an `item_id`.

## Does Not Own

| Not owned | Belongs to |
|---|---|
| The *meaning* of an external identifier inside its source system (e.g. what Shopify does with a variant ID) | The external system / integration worker |
| Assignment of external identifiers (Shopify assigns variant IDs; the supplier assigns their article numbers) | The external system. Whether the Item Domain itself *generates* any business identifier such as an internal article number is **open** (`OQ-ID-08`). |
| Application-local IDs for workflow records (task IDs, listing IDs) | Applications. These are not item identifiers and must not be registered as such. |
| Location or party identity | [Location](05-location-domain.md), [Party](06-party-domain.md) |

## Conceptual Model

```
Item
    item_id            ← canonical, opaque, stable

ItemIdentifier
    id                 ← identity of the identifier record itself
    item_id            ← the item this identifier resolves to
    namespace          ← which system / authority the identifier comes from   e.g. shopify, supplier, legacy_manager, internal
    identifier_type    ← what kind of identifier within that namespace        e.g. variant_id, product_id, article_number, sku
    value              ← the identifier value as a string                     e.g. "482901284"
    source             ← how / from where this record was established         (meaning vs. namespace is open, OQ-ID-05)
    metadata           ← namespace-specific extra data                        (content open, OQ-ID-05)
```

Example from the brief:

```
item_id:          itm_123
namespace:        shopify
identifier_type:  variant_id
value:            482901284
```

### Semantics

- An identifier is meaningful **only as the triple** `(namespace, identifier_type, value)`. The value `482901284` alone means nothing; `shopify/variant_id/482901284` does. (`INV-ID-03`)
- `namespace` answers *"who issues this kind of identifier?"*; `identifier_type` answers *"which of that issuer's identifiers is this?"*. A namespace may have several identifier types (Shopify has `product_id` and `variant_id`).
- Whether namespaces and types are a **closed, governed registry** or free-form strings is open (`OQ-ID-04`). Given that resolution rules depend on them, a governed registry is the **proposed** direction.

### Canonical `item_id`

- **CONFIRMED:** stable, unique, assigned by the Item Domain, referenced by all applications.
- **PROPOSED:** opaque (carries no business meaning), immutable, never reused after an item is retired/merged. The brief's examples use a prefixed form (`itm_123`); the actual format is open (`OQ-ID-07`).
- An `item_id` is **not** an `ItemIdentifier`. It must not be registered in an `internal` namespace as if it were an external identifier.

## Invariants

Full list in [11-invariants.md](11-invariants.md#identifiers).

**Confirmed**

- `INV-ID-01` — External/business identifiers are never the canonical identity.
- `INV-ID-02` — An `ItemIdentifier` record points at exactly one item.
- `INV-ID-03` — An identifier resolves according to its `(namespace, identifier_type)` semantics; a bare value is never resolved without them.

**Proposed**

- `INV-ID-04` — Within a `(namespace, identifier_type)`, a `value` resolves to **at most one** item. This is the proposed *default* uniqueness rule; it is the rule that makes resolution unambiguous. It is **not confirmed** because the business may have identifiers that are legitimately shared across items (see `OQ-ID-03`).
- `INV-ID-05` — Identifiers are added and removed only through Item Domain commands, which emit `ItemIdentifierAdded` / `ItemIdentifierRemoved`.

**Open**

- `INV-ID-06` — Whether an item may carry more than one identifier of the same `(namespace, identifier_type)`.
- Exact uniqueness scope (`OQ-ID-01`).

## Commands / Operations

| Command (conceptual) | Notes |
|---|---|
| `AddItemIdentifier { item_id, namespace, identifier_type, value, source?, metadata? }` | Validated against uniqueness rules (open) and namespace/type registry (open). |
| `RemoveItemIdentifier { item_id, identifier_id }` or by triple | Whether removed identifiers are retained as history is open (`OQ-ID-06`). |
| `ReassignItemIdentifier` (move an identifier from item A to item B) | **Not defined.** Needed if duplicates are ever merged (`OQ-ITEM-04`, `OQ-ID-06`). Listed so the design session considers it; do not assume it exists. |

Identifiers may also be supplied as part of `CreateItem`.

## Queries

| Query | Purpose |
|---|---|
| `ResolveIdentifier(namespace, identifier_type, value) → item_id` | The core query. Used by Scanner, integrations and every application during migration. Must be able to return **not found** and — depending on `OQ-ID-01` — **ambiguous**. |
| `ListIdentifiers(item_id)` | Show all known identifiers for an item. |
| `ResolveIdentifiers([...]) → [...]` (batch) | Bulk resolution for imports / projections. |
| `FindItemsByIdentifierValue(value)` across namespaces | Convenience for humans (Manager app search). Not a resolution primitive; results are inherently ambiguous. **Proposed**, not required. |

## Events

| Event | Payload (minimum) |
|---|---|
| `ItemIdentifierAdded` | `item_id`, `version`, `namespace`, `identifier_type`, `value` |
| `ItemIdentifierRemoved` | `item_id`, `version`, `namespace`, `identifier_type`, `value` |

Consumers such as the Shopify integration worker may key their own mappings off these events.

## Relationships

| Counterpart | Relationship |
|---|---|
| Applications | Each existing application holds identifiers in some namespace (its own legacy IDs, article numbers). During migration these are registered as `ItemIdentifier`s so the app can resolve to `item_id`. See [10](10-application-integration.md). |
| Shopify integration | Shopify product/variant IDs are identifiers in the `shopify` namespace. Which side is authoritative for creating the link is open (`OQ-MIG-04`). |
| Scanner | Resolves scanned codes to `item_id`. |
| Inventory | Inventory references `item_id` only — it never stores or resolves external identifiers. |

## Example Flow

### Shopify sync worker receives a product update

1. Worker receives a webhook for Shopify variant `482901284`.
2. Worker calls `ResolveIdentifier(shopify, variant_id, "482901284")`.
3. Result `itm_123` → worker proceeds using the canonical ID (e.g. issues an Item command, or updates its own sync bookkeeping).
4. Result *not found* → the worker's behaviour is a **business decision** (create an item? queue for manual matching? ignore?). Not specified — see `OQ-MIG-04`.

### Migration of the Manager app's legacy IDs (illustrative)

1. For each Manager-app item record, a canonical item is created (or matched to an existing one — matching rules open, `OQ-MIG-02`).
2. `AddItemIdentifier { namespace: legacy_manager, identifier_type: item_id, value: "<old id>" }` is issued.
3. The Manager app can now resolve its old IDs to `item_id` lazily, and store `item_id` on new records, without rewriting its history in one go.

## Failure / Conflict Cases

| Case | Behaviour |
|---|---|
| `AddItemIdentifier` with a triple already attached to *another* item | **Conflict** under the proposed rule `INV-ID-04`. Whether this is always an error, or allowed for some namespaces, is `OQ-ID-01` / `OQ-ID-03`. |
| `AddItemIdentifier` with a triple already attached to the *same* item | Idempotent no-op (proposed). |
| `AddItemIdentifier` with a second identifier of the same `(namespace, type)` on the same item | Depends on `OQ-ID-02`. |
| Resolution finds more than one item | Only possible if uniqueness is not enforced; the API must then be able to say **ambiguous** rather than pick one silently. |
| Unknown namespace / type | Rejected if a governed registry is adopted (`OQ-ID-04`). |
| Value normalization (`"A-4932"` vs `"a-4932"` vs `" A-4932 "`) | Open (`OQ-ID-09`). Must be decided per namespace/type before uniqueness can be enforced reliably. |

## Established Decisions

- Every item receives a stable canonical `item_id`; applications reference that ID.
- External/business identifiers (article number, SKU, Shopify product/variant ID, supplier article number, legacy IDs, others) are **not** canonical identity.
- They are modelled as `ItemIdentifier` records with `namespace`, `identifier_type`, `value`, `source`, `metadata`, attached to an `item_id`.
- The purpose of this model is to let existing systems resolve their identifiers to `item_id` **without** requiring immediate migration of every application's historical identifiers.
- Identifier uniqueness rules are recognized as important and are **deliberately not finalized** here.

## Open Questions

See [12-open-questions.md — Identity](12-open-questions.md#identity).

- `OQ-ID-01` — Uniqueness scope of `(namespace, identifier_type, value)`.
- `OQ-ID-02` — Multiple identifiers of the same `(namespace, type)` on one item.
- `OQ-ID-03` — Same identifier legitimately attached to multiple items (e.g. one supplier article number, many physical units). Strongly coupled to `OQ-ITEM-01`.
- `OQ-ID-04` — Governed namespace/type registry vs free-form.
- `OQ-ID-05` — Meaning of `source` vs `namespace`; content of `metadata`.
- `OQ-ID-06` — Identifier lifecycle: removal history, reassignment between items.
- `OQ-ID-07` — `item_id` format.
- `OQ-ID-08` — Does the Item Domain generate any business identifiers itself (e.g. an internal article number)?
- `OQ-ID-09` — Value normalization and case sensitivity per namespace/type.
