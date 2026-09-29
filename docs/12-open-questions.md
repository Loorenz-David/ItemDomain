# 12 — Open Questions

> **Purpose:** Every architectural decision that still has to be made, discovered while formalizing the architecture. Nothing here is answered; where the documents propose a default, it is labelled as a proposal and the question stays open until explicitly decided.
>
> **Timing legend:** **Before implementation** = must be decided before the affected component is built, because the answer changes its shape. **Deferrable** = can be decided later without rework, provided the initial design does not foreclose it.

Each entry: **Decision** · **Why it matters** · **Affects** · **Timing**.

---

## Item

### OQ-ITEM-01 — Item granularity — **RESOLVED**
- **Decision:** ~~Is an Item a product-level definition, a unique physical unit, or both?~~
- **Answer:** **Neither — an Item is a canonical business identity.** It is the identity the business assigns to one or more physical units currently treated as the same thing for identification purposes. It may stand for a single unique vintage piece or for sixty interchangeable chairs; the Item Domain does not distinguish the two cases and does not represent how many units there are.
- **Quantity is exclusively Inventory's.** Item answers *what canonical identity are these units represented as*; Inventory answers *how many exist and where* (`OQ-INV-01`, `INV-ITEM-14`).
- **No invariant that one Item = one physical unit.** One Item per unique vintage piece is a common usage pattern in this business, not a domain rule.
- **No product-definition or grouping semantics**, and no relationships between items at all (`OQ-ITEM-07`).
- **Consequences:** the two business identifiers are lifecycle stages rather than parallel schemes (`OQ-CID-01`); grouping similar pieces is deferred to `catalogue_id` (`OQ-ITEM-11`); changing an identity decision after the fact is the future marriage/divorce machinery (`OQ-ITEM-07`); how the grouping decision is made is `OQ-ITEM-12` (resolved: by the business, at registration).
### OQ-ITEM-02 — Is `name` a first-class canonical attribute? — **RESOLVED**
- **Decision:** ~~Is `name` a first-class attribute, a property definition, or not canonical at all?~~
- **Answer:** **Neither `name` nor `description` exists in this implementation.** A name carries no meaning the domain can validate and adds no value: identity is already `article_number` / `sku`, what the thing *is* is already classification plus properties, and what it *looks like* is already images.
- **Consequence:** no `name` or `description` on the Item, in events, or in projections. If the business later needs a model or range name ("Stockholm"), it becomes a `PropertyDefinition` scoped to the item types that have one — classification-driven and validated, not a universal free-text column. How humans recognise a piece instead is `OQ-ITEM-09` (resolved); whether any descriptive text is canonical at all is `OQ-PROP-06`.
### OQ-ITEM-03 — Dimensions and weight — **RESOLVED**
- **Decision:** ~~Are dimensions and weight universal first-class attributes or classification-driven properties?~~
- **Answer:** **Universal first-class canonical attributes.** Every physical object has dimensions and a weight regardless of what kind of thing it is, so they belong to the item itself.
- **The rule this establishes:** *universal physical facts are first-class attributes; type-conditional facts are properties.* Properties are conditional attributes of an item **given its category and type** — a seat count means nothing for a lamp. This is the test for any future candidate field.
- **Universal does not mean mandatory.** They are nullable at creation and filled in when the Worker measures the piece after intake (`OQ-ITEM-06`). "Universal" describes what kind of fact it is, not when it must be present.
- **Units — RESOLVED:** **millimetres for every dimension, grams for weight**, fixed domain-wide. Small units were chosen deliberately. The unit is **not stored per value**; it is carried in the field name — `height_mm`, `width_mm`, `depth_mm`, `weight_g` — so a bare `220` can never be mistaken for centimetres.
- **Conversion is the clients' job.** Applications and frontends read the unit from the field name and are responsible for converting whatever a person enters or sees (cm, m, kg) to and from mm and g, and for protecting the values they send. The Item Domain accepts and returns mm and g only; it offers no unit conversion.
- **Resolved later:** each dimension and the weight is optional on its own (an item may have only some), and a value that is present must be **greater than zero** (`INV-ITEM-24`). They can be cleared back to none (`INV-ITEM-25`).
- **Precision — RESOLVED:** **whole numbers only.** That is why the small units (mm, g) were chosen.
### OQ-ITEM-04 — Item lifecycle: deletion, retirement, merging — **RESOLVED**
- **Decision:** ~~Can items be deleted? Retired? Merged? Is there a canonical lifecycle status?~~
- **Answer:**
  - **Deletion: yes, and every deletion is a soft delete.** The record is retained and flagged; nothing is physically removed.
  - **Retirement: does not exist.** No retire operation, no retired status.
  - **Merging: does not exist.** Two item records are never fused. If the business has more of a thing than the record says, that is a **quantity** fact in the Inventory Domain — `receive` or `adjust` at a location — not an operation on the item record.
  - **Deletion does not consult Inventory.** The Item Domain never reads Inventory state, so the forbidden `Item → Inventory` direction stays forbidden and an item may be soft-deleted while positions still exist.
- **Consequences:** `article_number` and `sku` are permanently unique — a soft-deleted item keeps its values and they are never reused (resolves half of `OQ-CID-05`). Migration cannot lean on merging to reconcile duplicates, which raises the stakes on `OQ-MIG-02`. Remaining sub-questions — query visibility, undelete, phantom stock — are `OQ-ITEM-10`.
### OQ-ITEM-05 — Aggregate and version boundary — **RESOLVED**
- **Decision:** ~~Do identifier and property changes increment `item.version`?~~
- **Answer:** **Yes — that is the point of the version.** Property changes and identifier changes both increment `item.version`, which tracks the item's canonical state as a whole. **By the same rule image changes increment it too**, images being canonical state (`INV-IMG-01`).
- **Consequence — the contention is accepted deliberately.** Two applications editing different parts of the same item will conflict, and that is intended: the alternative is silent divergence. In practice a Worker uploading eight photographs produces eight version bumps, any of which can invalidate a concurrent edit elsewhere.
- **What this obliges:** applications need a **re-read-and-retry path**, not a one-shot write — a conflict is an expected outcome on a busy item, not an exceptional error. It also raises the stakes on command granularity (`OQ-API-02`): a coarse `UpdateItem` saving a whole form is one bump where fine-grained commands are many.
### OQ-ITEM-06 — Mandatory fields at creation — **RESOLVED**
- **Decision:** ~~What must be present to create an item?~~
- **Answer:** `article_number`, `item_type` (which determines the category — `OQ-CLS-05`), and all **required properties** for that classification. **Nothing else is mandatory.**
- **Consequence for `OQ-PROP-03`:** required properties are enforced **at creation**, which settles the "when" half of that question — and *sharpens* the other half rather than softening it. An item created legitimately before a property definition became `required` would be retroactively invalid, and nothing yet says what happens then.
- **Consequence for dimensions and weight:** universal canonical attributes (`OQ-ITEM-03`) but **not** mandatory at creation — the Worker measures the piece after intake. They are nullable at creation and filled in later, the same shape as `sku`. Do not make them `NOT NULL`.
- **Note:** the category is not a separate input. The item type determines it, and `item.category_id` is not stored (`OQ-CLS-05` resolved).
### OQ-ITEM-07 — Item-to-item relationships — **RESOLVED**
- **Decision:** ~~Are there variants, sets, components, or "same model as" relationships?~~
- **Answer:** **None.** An Item is an independent canonical row with **no relationships to other Items**. Do not model variants, sets, components, parent/child items, or any generic item-to-item relationship.
- **Commercial grouping is not a relationship.** Deciding to restore and sell four chairs as a set is a commercial act, not a persistent link between item rows.
- **Splitting and grouping identity is separate machinery — "marriage" and "divorce".** The business may later decide that units currently under one identity should become separately identified, producing **new Item identities with their own `item_id`, `article_number` and potentially `sku`**. The inverse, grouping identities together, may exist too.
- **Worked example:** Item A carries article number `CHAIR-001`; Inventory records quantity 60. Thirty of those chairs arriving at the warehouse changes **Inventory quantity and location — not Item identity**. If four are later restored and sold as one separately identified group while the rest stay under the original identity, that is an identity split, and it goes through the future marriage/divorce machinery.
- **Do not design the mechanism yet.** How quantities move, how new identities are created, whether identifiers are inherited or regenerated, how history is represented, the transactional boundaries, and which domain or application owns the operation will all be defined later. For now record only that it exists, is out of scope, and **must not be pre-modelled** (`INV-ITEM-21`).
- **Working assumption:** items are created with a valid, pure identity; that identity stays persistent; the Item Domain models no relationships between items.
- **Not a contradiction with `OQ-ITEM-04`:** "no merge" there and "marriage exists" here are compatible. There is no merge *in this implementation*; marriage/divorce is separate, later machinery outside this domain's current scope.
### OQ-ITEM-08 — Images / media — **RESOLVED**
- **Decision:** ~~Are photos canonical item data, application data, or a separate concern?~~
- **Answer:** **Canonical.** An item's images are part of its canonical description — the visual answer to "what is this thing?". Modelled as `ItemImage` link records holding an absolute `url` and a storage/source `type` (since refined: `position`, `uploaded_by`, `storage_type`, `storage_key`, `access_type` — `OQ-IMG-01`, `04`, `05`). The Item Domain owns the references; the image files stay in application or third-party storage.
- **Consequence:** photography *workflow* stays with the Worker and Seller applications (`INV-IMG-06`), and a new cluster of questions opens — see [Images](#images).

### OQ-ITEM-09 — How is an item presented to a human? — **RESOLVED**
- **Decision:** ~~With no `name`, what does an application show in a list, a task card or a scanner screen?~~
- **Answer:** **the item type, the image, and the article number or SKU.** That is what a human needs to recognise a piece. Neither `name` nor `description` exists (`OQ-ITEM-02`).
- **Consequence:** projections cache the item type (and category), the image, and `article_number` / `sku`. This **raises the practical weight of `OQ-IMG-01`**: if a human identifies a piece partly by its picture, the model has to say *which* picture is shown when only one fits — and that question is currently deferred.
### OQ-ITEM-10 — Soft-delete semantics — **RESOLVED (mechanism), partially open**
- **Answer:** implement soft deletion **as the industry standard defines it**:
  - a nullable deletion marker (`deleted_at`) on the row; nothing is ever physically removed;
  - deleted rows are **excluded from queries by default**, including `GetItemByArticleNumber`, SKU lookup and external identifier resolution;
  - callers that need them — audit, administration, migration tooling — **opt in explicitly**;
  - **restore is possible**: clearing the marker undeletes the item — a `RestoreItem` command emitting `ItemRestored`. It is the **only** change a deleted item accepts (`INV-ITEM-26`), and it never conflicts on `article_number` / `sku`, because those are never reused;
  - deletion does **not** cascade to other domains.
- **Note:** this does *not* relax permanent uniqueness. A soft-deleted item keeps its `article_number` and `sku` and they are never reused (`INV-CID-06`) — stricter than many soft-delete implementations, and deliberate.
- **Still open, because the industry standard does not address them:**
  - **projection behaviour** — on `ItemDeleted`, should a consumer drop its row or flag it? Dropping loses context an application may still need for its own already-closed workflows.
  - **phantom inventory** — deletion deliberately does not consult Inventory (`INV-ITEM-13`), so positions can outlive a deleted item and nothing surfaces them. Who reconciles, and how often?
- **Timing:** the residuals are **deferrable**; the mechanism is settled.
### OQ-ITEM-12 — What makes two physical things one Item identity, or two? — **RESOLVED**
- **Decision:** ~~On what basis does the business decide that two physical objects share one canonical Item identity rather than take separate ones?~~
- **Answer:** **It is a business decision made during registration and management of the items — not something the domain infers.** The rule is *not* that the Item Domain determines algorithmically whether two physical things deserve the same identity. The domain records and enforces the resulting identity; it never evaluates similarity.
- **Semantic formulation:**

  > An Item is the canonical identity assigned by the business to **one or more** physical units that are currently treated as the same thing for identification purposes. The number of units represented by that identity is exclusively an Inventory concern.

- **What this settles:**
  - There is **no invariant that one Item equals one physical unit** (`INV-ITEM-14`).
  - Multiple physical units may share one `item_id`, one `article_number` and one `sku` (`INV-ITEM-20`).
  - One Item per unique vintage piece is a **business usage pattern, not a domain rule**, and nothing in the model may assume it.
  - The domain neither infers nor validates the grouping decision (`INV-ITEM-22`).
- **Deliberately out of scope:** the criteria and process by which a human decides to group or split units stay outside the Item Domain specification, and may later be formalized as part of marriage/divorce (`OQ-ITEM-07`).
### OQ-ITEM-13 — Wish list: pieces the business wants but does not own — **RESOLVED**
- **Decision:** ~~Where do pieces the business wants but neither owns nor holds live?~~
- **Answer:** **in their own application, implemented later.** A wanted piece is not an Item and not inventory; it becomes an Item only when it is acquired (and gets its `article_number` then).
- **Consequences:** `article_number` keeps its meaning — "we have it and are tracking it" (`OQ-CID-01`). Inventory keeps recording only stock the business owns or has sold (`INV-INV-13`), so "do we have it?" stays reliable. The wish-list app may reuse item types and property definitions as search criteria, and must not store its entries as Items or inventory.

### OQ-ITEM-11 — `catalogue_id`: grouping similar pieces (future concept)
- **Status:** **Explicitly deferred — not part of this implementation.** Recorded here so it is not reinvented under another name, and not approximated by something else in the meantime.
- **The concept:** a `catalogue_id` would group similar Items by the core properties that define them, potentially using similarity or vectorization machinery. It is a *grouping over* canonical identities, never a replacement for them — `INV-ITEM-14` holds regardless. It is also distinct from marriage/divorce, which *changes* identity rather than grouping over it.
- **What this means now:** build no grouping, no product-definition record, no shared parent, no "variants", and no field that half-does it.
- **What it will need later:** a nullable reference on the item plus a new entity, and a decision about which properties count as "core". Nothing in the current model has to anticipate it, so deferring costs nothing.
- **Timing:** **Deferred.** Revisit when the business asks for grouping, not before.

## Identity

Identity has three layers (see [02](02-canonical-identity-and-identifiers.md)). The questions below are grouped accordingly: `OQ-CID-*` concern the canonical business identifiers we issue ourselves; `OQ-ID-*` concern foreign-issued external identifiers and the surrogate key.

### OQ-CID-01 — What distinguishes `article_number` from `sku`? — **RESOLVED**
- **Decision:** ~~What is the business difference between the two canonical business identifiers?~~
- **Answer:** They mark **different points in a single piece's life**:

  | Identifier | Assigned when | Says |
  |---|---|---|
  | `article_number` | the piece is created / registered at intake | we have it and are tracking it |
  | `sku` | the restored piece is published for sale | it is being offered |

- **Consequences:** `article_number` is required at creation; `sku` is normally absent until publication, though it may be sent at creation (`OQ-CID-02` resolved, `INV-CID-07`). Lookups and projections must tolerate items with no `sku`. **A missing `sku` must not be read as listing status** — the Item Domain has no concept of publishing and must never gain one; listing state stays with the Seller application (`INV-ITEM-02`).
### OQ-CID-02 — Are `article_number` and `sku` mandatory? — **RESOLVED**
- **Decision:** ~~Must both be present at creation? Can an item have one but not the other?~~
- **Answer:** **`article_number` yes, `sku` no.** A piece is registered with an article number; the SKU is assigned later, when the restored piece is published for sale. An item therefore exists in a perfectly valid state with no `sku`.
- **Consequences:** `sku` is nullable, and every query, projection and integration must handle its absence. Uniqueness on `sku` is enforced only across items that have one. Note this does **not** reopen stub creation — classification (`OQ-ITEM-06`) and `article_number` are both still required at creation.
### OQ-CID-03 — Format and validation rules — **RESOLVED**
- **Decision:** ~~What formats are legal for each (`A-4932`-style patterns, length, allowed characters)? Who defines and maintains the scheme?~~
- **Answer:** **No format restriction for now.** `article_number` and `sku` are **strings**, like every identifier in the system. The domain enforces uniqueness and normalization (`OQ-CID-05`), not a pattern.
- **Consequence:** the domain cannot reject a malformed number, only a duplicate one. A pattern can be added later, but values accepted before it exist permanently, because they are never deleted and never reused.
- **Empty values are rejected** — including a value that is empty after trimming. An identifier nobody can reference is pointless. This applies to external identifier values too.
- **No maximum length.**

### OQ-CID-04 — Are they mutable? — **RESOLVED**
- **Decision:** ~~May an `article_number` or `sku` be corrected or reassigned after the fact? If so, is the old value retained, and can it be reused?~~
- **Answer:** **Yes, both can change**, so typos can be corrected. **The old value is kept** as a former value of that item, and **lookups by `article_number` or `sku` match former values as well as current ones.** A label printed with the typo therefore still finds the item.
- **Consequences:**
  - **Uniqueness covers current and former values.** A former value of item A still resolves to A, so no other item may ever take it. This is the permanent uniqueness of `INV-CID-06` extended to history.
  - **Lookups stay unambiguous.** A value matches at most one item, whether as its current or a former value.
  - **The item needs a history of its business identifiers** — per item: which identifier (`article_number` or `sku`), the former value, and when it was replaced. This is part of the item aggregate: a change increments `item.version` and emits an event carrying the old and new values.
  - **A lookup should tell the caller when it matched a former value**, so an app can show "label outdated, current number is …" instead of silently carrying on.
- **Resolved later:** `sku` can be cleared once set; the cleared value is kept as a former value (`INV-ITEM-25`).
- **Going back is allowed:** an item may take one of its **own** former values again. It is recorded as a **new change** (a new version, a new event, a new history entry), just like any other value going back — e.g. a height changed from 100 to 101 and then back to 100. The value becomes current again; the value it replaces becomes a former value. It is still never available to another item.

### OQ-CID-05 — Uniqueness scope and normalization — **RESOLVED**
- **Resolved:** uniqueness is **permanent and global**. A soft-deleted item keeps its `article_number` and `sku`, and those values are never reused by another item (`OQ-ITEM-04`). Former values are covered too (`OQ-CID-04`).
- **Answer — normalization before storing:** leading and trailing spaces are **trimmed**, the value is **lowercased**, and each run of spaces inside it becomes a single `-`. `" A 4932 "` is stored as `a-4932`.
- **Matching ignores the separator.** Lookups and uniqueness both compare values with `-` removed, so searching `A 4932`, `a-4932` **or `a4932`** finds the item stored as `a-4932`.
- **Consequence — `a4932` and `a-4932` are the same identity.** Once one exists, the other is rejected as a duplicate. This is what keeps a separator-less search unambiguous.
- **Consequence:** the stored value is the normalized one; the original spelling is not kept. Labels printed from the domain show the lowercase, hyphenated form.
- Leading zeros are significant: the values are strings.
### OQ-ID-01 — Uniqueness scope of identifiers — **RESOLVED**
- **Decision:** ~~Is `(namespace, identifier_type, value)` unique across all items?~~
- **Answer:** **No. The same external value may be attached to several items**, for every identifier type. External apps may use the identifier table to group canonical items for their own reasons — in effect, **a way for apps to put their own tags on canonical items**.
- **Consequences:**
  - resolving an external value returns a **list** of items (possibly one, possibly none) — never a silently picked single item;
  - the Item Domain gives these groupings **no meaning**: they are the app's tags, not item-to-item relationships (`INV-ITEM-15`) and not `catalogue_id` (`OQ-ITEM-11`);
  - attaching the same value, type and source to the same item twice is a no-op.

### OQ-ID-02 — Multiple external identifiers of the same type on one item — **RESOLVED**
- **Decision:** ~~May an item carry two `shopify/variant_id`s (e.g. listed twice), or two legacy IDs from the same system?~~
- **Answer:** **Yes.** An item can carry several identifiers of the same type. The main case: an item listed in **several Shopify shops** has one Shopify ID per shop.
- **Consequence:** `source` says which shop or app each identifier belongs to (`OQ-ID-05`), so an integration picks the one for *its* shop by `source`, not by "the current one".
- **Still open (small):** may one item carry two identifiers of the same type **from the same source** (e.g. listed twice in the same shop)?

### OQ-ID-03 — Same external identifier on multiple items — **RESOLVED**
- **Answer:** yes, for every identifier type (`OQ-ID-01`). Resolution returns a list.
### OQ-ID-04 — Governed namespace/type registry vs free-form — **RESOLVED**
- **Decision:** ~~Is the set of namespaces and identifier types a closed list managed in the Item Domain, or can callers invent them?~~
- **Answer:** **Governed by the Item Domain, stored in a separate table** (`ExternalIdentifierType`). An `ItemIdentifier` must reference an existing type; unknown types are rejected. **Users can add new types** when a new kind of external ID has to be accepted — it is reference data, not a fixed code list.
- **Consequence:** adding a type is a reference-data command, so who may add one falls under `OQ-CLS-02` / `OQ-AUTHZ-03`. The uniqueness rule of `OQ-ID-01` is a natural attribute of that table.

### OQ-ID-05 — Meaning of `source` and content of `metadata` — **RESOLVED**
- **Decision:** ~~What does `source` capture that `namespace` does not? What may go in `metadata`?~~
- **Answer — `source`:** the **specific system the identifier comes from**: `shopify store x`, `manager app`, `purchase app`. The namespace says *which kind* of system issued the ID (Shopify); the source says *which one* (store x). This is what tells two Shopify IDs on the same item apart (`OQ-ID-02`).
- **Answer — `metadata`: removed.** No concrete use was found for it, and a field whose content is undefined becomes a dumping ground the domain cannot validate. Data that only an integration needs (a Shopify handle, an admin URL) stays in that integration. If a real need appears later, it is added as a named, typed field.
- **Answer — where `source` comes from:** a governed **`Source` table** of registered sources. **When an app registers for an API key, it registers itself as a source.** The API key therefore tells which source a request comes from: the authentication layer resolves the key and puts the source into a **request context object**, and every `source` field is filled from that context — **never from the request payload**.
- **Consequences:**
  - a source cannot be mistyped or spoofed in a payload;
  - the same context fills `ItemImage.uploaded_by`, `LocationIdentifier.source` and the history table's `source_id`;
  - **Registration carries a unique name.** The request that registers a source for an API key carries a **name / reference** chosen by the external app (e.g. `shopify-store-x`, `manager-app`). Names are **unique** among sources, so a name identifies exactly one source.
  - **One registration per source that must be told apart.** Two Shopify shops are told apart by `source` (`OQ-ID-02`), so each shop registers under its own name with its own key (`shopify-store-x`, `shopify-store-y`) — even if one integration worker serves both and holds both keys. A single registration for the worker would give both shops the same source.
  - **The Shopify proxy.** Shopify is reached through a **Shopify app proxy**, a central system of its own that acts **on behalf of** the shops. **Each shop is its own external application**: the proxy stores an individual API key per shop and uses that shop's key for every request it makes for that shop. The author of every change — the shop — therefore stays identifiable in identifiers, images and the history table.

### OQ-ID-06 — External identifier lifecycle
- **Decision:** On `RemoveItemIdentifier`, is history retained (so an old Shopify ID can still be traced)? Can an identifier be reassigned from one item to another?
- **Why it matters:** Traceability during and after migration. Reassignment is **not** needed for merging, which does not exist (`OQ-ITEM-04` resolved), but it may still be needed to move a Shopify or legacy identifier off a soft-deleted duplicate onto the surviving item.
- **Affects:** Identifier model, events.
- **Timing:** **Deferrable**, unless duplicate handling during migration needs it.
### OQ-ID-07 — `item_id` format — **RESOLVED**
- **Decision:** ~~UUID, prefixed string (`itm_…`), other? Generated by the domain only?~~
- **Answer:** **a prefixed string, `itm_…`**, minted by the Item Domain only. The prefix makes an item ID recognisable wherever it appears — logs, URLs, other domains' records — and cannot be confused with a `loc_…` or a business identifier.
- **Suffix:** the **record's own unique id**, which is a **UUIDv7** — `itm_` + the item row's UUIDv7.
- **Why UUIDv7 and not a number:** a number reveals how many items exist and how fast they grow, can be guessed, must be handed out by the database, clashes when data from several databases is combined, and invites code to rely on its order. UUIDv7 has none of these costs, and because it is time-ordered it inserts into indexes almost as smoothly as a number. Humans refer to pieces by `article_number` / `sku`, so readability of `item_id` matters little.

### OQ-ID-08 — Does the Item Domain generate business identifiers? — **RESOLVED**
- **Decision:** ~~Does the domain *issue* article numbers / SKUs, or only *record* them?~~
- **Answer:** **No.** Values for `article_number` and `sku` are **supplied** by the creating application or person. The Item Domain enforces uniqueness and format but does not generate them.
- **Consequence:** The domain owns no numbering sequence. It enforces uniqueness and normalization; there is no format rule for now (`OQ-CID-03`). An item exists without an `sku` until publication (`OQ-CID-02`).

### OQ-ID-09 — External value normalization — **RESOLVED**
- **Decision:** ~~Case sensitivity, whitespace, leading zeros, per namespace/type.~~
- **Answer:** **No normalization.** The external system decides how its IDs look, so the Item Domain stores the value **exactly as given** and matches it exactly. It must not interfere with a value it does not own.
- **Consequence:** this is the opposite of our own identifiers (`OQ-CID-05`), deliberately: we control our numbering, not theirs. Callers must send the value exactly as the external system issued it; `ABC-1` and `abc-1` are different values.

### OQ-ID-10 — Where other identities are stored — **RESOLVED**
- **Decision:** ~~Confirm that a supplier's article number is an external identifier, not canonical identity.~~
- **Answer:** `article_number` and `sku` are the only canonical business identities. **Any other identity an item has is stored as an external identifier.** The "supplier article number" example has been removed from the documents: it is not part of this system and it read too much like our own `article_number`.

## Classification

### OQ-CLS-01 — Hierarchy depth
- **Decision:** Fixed two levels (Category → ItemType) or deeper/variable?
- **Why it matters:** Property scoping rules multiply with depth.
- **Affects:** Reference-data model, `allowed(item)` computation.
- **Timing:** **Deferrable** if two levels are adopted now and the model does not forbid a parent on Category later.

### OQ-CLS-02 — Reference-data governance — **RESOLVED**
- **Decision:** ~~Who may create/change categories, item types and property definitions? Through which application (Manager?) and with what review?~~
- **Answer:** **Anyone — any application or user.** There is no restricted role and no review step.
- **Control is by traceability, not permission:** every change to reference data is **recorded in the polymorphic history table** — a single history table that records changes to any kind of entity (entity type + entity id), so a category, an item type or a property definition can be traced back: what changed, when and by whom.
- **Consequences:**
  - Because a change can make existing items non-conforming, the review happens **after** the change, through the dedicated resolution endpoints of `OQ-PROP-03`.
  - Moving an `ItemType` to another category (`OQ-CLS-05`) is allowed and recorded like any other change.
  - This answers the classification part of `OQ-AUTHZ-03`, and the "who supplies them" part of `OQ-IMG-07`.
- **The history table — industry-standard polymorphic audit log.** One table records changes to **every** entity of the domain — items and their parts as well as reference data:

  ```
  History
      id
      entity_type        ← Item | Category | ItemType | PropertyDefinition | ExternalIdentifierType | …
      entity_id          ← the id of that entity (polymorphic: no database foreign key)
      action             ← created | updated | deleted | restored
      changes            ← per changed field: old value and new value (JSON)
      entity_version     ← the entity's version after the change, where it has one
      source_id          ← from the API key, via the request context
      user_name_snapshot ← as sent (OQ-AUTHZ-02)
      causation_id       ← the command's idempotency key, to link the entry to its event
      occurred_at
  ```

  - **Append-only:** entries are never changed or deleted. Retention: kept indefinitely (proposed, `OQ-OPS-01`).
  - Written **in the same transaction** as the change, like the outbox — a change without its history entry cannot commit.
  - Indexed on `(entity_type, entity_id, occurred_at)` for "show me this item's history".
  - It is the **permanent record**; the outbox is only for delivering events.

### OQ-CLS-03 — Reclassification semantics — **RESOLVED**
- **What each change to an item type does:**
  - **Rename a type:** only the type changes. Items point at the type by id, so no item is touched; an event tells applications that type *x* has a new name, and they update their cached names.
  - **Change a property definition of a type:** the definition changes, applications are told, and items that no longer conform appear in the resolution list (`OQ-PROP-03`).
  - **Create a type:** items can then be created with it, or moved to it.
  - **Delete a type that still has items:** the delete request **must name the type the items move to**, because an item must always have a type. The items move in the same operation.
  - **Change one item's type:** the same move, for one item.
- **What happens to property values on a move** (definitions live on types, so no value carries over by itself):
  - a value **carries over** when the new type has a definition with the **same name** and the value is valid for it (e.g. `red` moves if the new type's colour list contains `red`);
  - every value that cannot carry over is **recorded in the polymorphic history table and removed** from the item;
  - an item missing a value the new type **requires** goes to the **resolution list** (`OQ-PROP-03`) — the move is **not** blocked. This matters most when a type with many items is deleted.

### OQ-CLS-04 — Unclassified items
- See **OQ-ITEM-06**.

### OQ-CLS-05 — Is `item.category_id` redundant? — **RESOLVED**
- **Decision:** ~~Keep both `category_id` and `item_type_id` on the item (with INV-ITEM-06 enforcing agreement), or derive category from type?~~
- **Answer:** **Derive it.** The item stores only `item_type_id`; its category is always `item_type.category_id`. Redundant data can disagree; derived data cannot.
- **Consequences:**
  - `INV-ITEM-06` (type belongs to the item's category) now holds **by construction** — there is no second value to disagree with, so there is no validation error for it.
  - `CreateItem` and `ClassifyItem` take `item_type_id` only; they no longer accept a category.
  - It relies on every `ItemType` belonging to exactly one `Category` (`INV-CLS-03`, now confirmed).
  - **Moving an `ItemType` to another category recategorises every item of that type at once**, with no item version bump and no item event. Whether that reference-data change is allowed at all falls under governance (`OQ-CLS-02`).
  - Filtering items by category is a join through the item type, and projections that show the category must resolve it from the type.

## Properties

### OQ-PROP-01 — Scope precedence — **RESOLVED**
- **Decision:** ~~When a property is defined at category level and again at item-type level, which wins?~~
- **Answer:** **Property definitions live only on item types.** There are no category-level definitions, so a property can never be defined twice for one item and there is no precedence to decide. `Sofa.color` and `Chair.color` are two separate definitions, each with its own list of choices.
- **Consequences:**
  - Picking a type gives a form exactly that type's definitions — users effectively build their own forms per type.
  - **Searching by property name without a type searches every definition with that name** (all `color` properties). Names therefore matter across types: `color` and `colour` would be two different properties to search.
  - A name is unique within one item type.

### OQ-PROP-02 — Data types, enums, multi-valued properties — **RESOLVED**
- **Decision:** ~~Supported `data_type` set; how enumerations are defined; whether a property may hold multiple values.~~
- **Answer:** five data types: **text, whole number, decimal, yes/no (boolean), and a choice from a fixed list**.
- **Choice lists:** the definition holds the list of values a user can pick from, plus whether **one or several** may be picked. Single selection is the normal case; multiple selection (e.g. `material = oak + leather`) is allowed **only** for choice-list properties.

### OQ-PROP-03 — Definition evolution and enforcement timing — **RESOLVED**
- **Decision:** ~~What happens to existing items when a definition changes (new `required`, tightened constraint)? Are definitions versioned? When is `required` enforced?~~
- **Answer:** **The change is allowed, and the affected items are surfaced for people to fix.** A dedicated set of endpoints and services lets users **list** the items that no longer conform to their definitions, **see** each one and what is wrong, and **resolve** it. A definition change thereby becomes a prompt to re-evaluate the items it concerns, not a silent break.
- **`required` is enforced at creation** (`OQ-ITEM-06`) and on every write; an existing item that became non-conforming because of a later definition change is **not** rejected retroactively — it shows up in the resolution list.
- **Consequence for the invariants:** `INV-CLS-01` / `INV-CLS-02` / `INV-CLS-05` are checked on every **write**. An item may be non-conforming **between** a definition change and its resolution, and that state is visible, not hidden.
- **Still open (small):**
  - how non-conforming items are found: computed when the list is queried, or recorded when the definition changes;
  - whether a write to an unrelated part of a non-conforming item (e.g. adding an image) is allowed, or must fix the issue first;
  - whether definitions are versioned (the history table records the change either way).

### OQ-PROP-04 — Free-form properties — **RESOLVED**
- **Decision:** ~~May an item carry a property with no definition?~~
- **Answer:** **No.** Every property value belongs to a definition of the item's type. Anyone can create a definition (`OQ-CLS-02`), so adding one properly is cheap.
- **Timing:** **Before implementation.**

### OQ-PROP-05 — Localization
- **Decision:** Are property names / enum values / category names localized?
- **Timing:** **Deferrable**.

### OQ-PROP-06 — Canonical descriptions vs channel copy — **RESOLVED**
- **Decision:** ~~Is there any canonical descriptive text, or is all descriptive text channel copy owned by Seller/Shopify?~~
- **Answer:** **No canonical title or description.** A listing's title and description belong to the sales channel (Seller/Shopify) that shows them.
- **Users may still add one as a property** — e.g. a `title` definition on a type. That is the user's choice about their vocabulary; the system neither requires nor treats it specially.
### OQ-IMG-01 — Ordering and primary image — **RESOLVED**
- **Decision:** ~~How does a consumer know which image to show when it can only show one, and in what order the rest appear?~~
- **Answer:** **an order column, `position`, on `ItemImage`.** Users can reorder an item's pictures. If the client (e.g. an external service) does not supply a position, the domain assigns it **from the list index** — the image's place in the list it arrived in.
- **Consequence:** the order is canonical, so every application shows the images in the same order, and the first image is the one shown when only one fits. Reordering is an item change: it increments `item.version` and emits an event.
- **Positions are managed by the domain and never have gaps.** They always run 1, 2, 3 …: removing an image closes the gap, and adding or moving an image to a position already taken shifts the following images down.

### OQ-IMG-02 — Role / purpose of an image
- **Decision:** Does an image record what it depicts — front, detail, damage, label, 360?
- **Why it matters:** Worker documentation photos and Seller listing photos serve different ends. Without a role, consumers can only infer purpose from ordering. Distinct from `type`, which records storage/source rather than content.
- **Affects:** `ItemImage` model, Worker, Seller/Shopify.
- **Timing:** **Deferrable.**

### OQ-IMG-03 — Alt text and captions
- **Decision:** Does an image carry descriptive text? Localized (see `OQ-PROP-05`)?
- **Why it matters:** Accessibility, and Shopify listings expect alt text. If it is not canonical, each channel writes its own.
- **Affects:** `ItemImage` model, Seller/Shopify.
- **Timing:** **Deferrable.**

### OQ-IMG-04 — URL durability and storage ownership — **PARTIALLY RESOLVED**
- **Resolved — the model:** an image carries an **absolute `url`**, and **may also carry a `storage_type` and a `storage_key`** from which the reference can be rebuilt. For now clients are **required to send absolute URLs**; the storage type + key capability exists in the model so that private storage (e.g. signed or expiring URLs) can be added later without a schema change.
- **Resolved — checking:** the backend **checks that the URL resolves when the image is attached**. The checker is built as a **standalone component**, detached from the attach command, so a background worker can reuse it later to re-check existing images.
- **Deferred:** the background re-check job, and who repairs a reference that has died since it was attached.
- **When the check fails, the image is kept and marked `state = not_available`** — not rejected. A later check that succeeds marks it `available` again. Apps can skip or flag unavailable images.

### OQ-IMG-05 — Semantics of `type` — **RESOLVED**
- **Decision:** ~~Is `type` the application that uploaded the image, or the storage backend the file lives in?~~
- **Answer:** **Both, and more — the single `type` is split into three fields**, because the domain needs to know three separate things:
  - `uploaded_by` — the **source** that uploaded the image, taken from the request context (`OQ-ID-05`);
  - `storage_type` — the **storage** the file lives in (e.g. a bucket or Shopify's CDN);
  - `access_type` — **how the file can be accessed** (for now: public absolute URL; later possibly private, via `storage_key` — `OQ-IMG-04`).
- **Consequence:** provenance and location no longer get confused when two apps write to one bucket, or one app moves its storage.
- **Still open (small):** the field names above are proposed; whether each set of values is a governed table (like `ExternalIdentifierType`, `OQ-ID-04`) or free text.

### OQ-IMG-06 — Aggregate membership and uniqueness — **RESOLVED**
- **Decision:** ~~Do image changes increment `item.version`? May the same URL be attached to more than one item? Is there a minimum or maximum count?~~
- **Answer:**
  - image changes increment `item.version` (`OQ-ITEM-05`);
  - **the same URL may be attached to several items** — each attachment is its own `ItemImage` record;
  - **there is no minimum**: an item may have no images at all.
- **Consequence:** apps must handle an item with no image, even though the image is one of the three things people recognise a piece by (`OQ-ITEM-09`). Removing an image from one item never affects another item using the same URL.
- **No maximum** number of images.

### OQ-IMG-07 — Category / ItemType presentational assets — **RESOLVED**
- **Decision:** ~~Are `icon_url` and `image_url` mandatory on every category and item type? What does an application render when one is missing? Who supplies them?~~
- **Answer:**
  - **one field, named the same on both: `icon_url`.** `image_url` is dropped from `Category` and `ItemType`;
  - **not required**: when missing, the API simply returns none (`null`). The domain defines no fallback;
  - anyone may supply or change it (`OQ-CLS-02`).
- **Consequence:** what an app draws in place of a missing icon is up to each app.

## Inventory

### OQ-INV-01 — Quantity semantics — **RESOLVED**
- **Decision:** ~~Arbitrary counts, or 0/1 per unique unit?~~
- **Answer:** **Counts — and quantity is the Inventory Domain's sole responsibility.** One Item identity may cover many units: sixty homogeneous chairs registered under one identity sit at quantity 60. A unique vintage piece sits at quantity 1 because the business registered one unit under that identity, not because any rule caps it.
- **The Item Domain never expresses multiplicity** and imposes no constraint on quantity (`INV-ITEM-14`, `INV-INV-09`).
- **Superseded:** an earlier reading recorded quantity as presence (0 or 1), derived from reading "one Item = one unique physical piece" as a rule. That is **withdrawn** — see `INV-INV-12`.
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

### OQ-INV-08 — Party references and ownership — **PARTIALLY RESOLVED**
- **Resolved — Inventory only ever points at locations** ("Option A"). Stock at a dealer, at a supplier or restorer, or with a customer is stock at a **location whose type says so** (e.g. `dealer`, `supplier`, `customer`). Every position is `item × location`; queries never split into "at a location" vs "at a party".
- **Consequence:** the location type is what tells the user and the system whether stock is ours and on our premises, ours but held by someone else (consignment, repair), or gone (sold). "Do we have it?" therefore depends on which location types count as ours.
- **Customers too:** a sale moves the piece to the **customer's location**, so "where is it?" answers "at customer X". "Do we have it?" is answered by which location types count as ours (`OQ-LOC-08`).

### OQ-INV-09 — Position persistence
- **Decision:** Are positions stored and kept in sync with movements, or derived from the movement log on read?
- **Why it matters:** Stored positions are fast to query but must be kept consistent with the movements that produced them (`INV-INV-05`); derived positions cannot drift but cost more to read and complicate concurrency.
- **Affects:** Inventory persistence, queries, concurrency mechanism (`OQ-CC-03`).
- **Timing:** Design session.

*(An earlier draft asked whether a position should collapse to "the piece's current location". That question assumed quantity was capped at 1; with quantity confirmed as an ordinary count, it no longer applies.)*
## Location

### OQ-LOC-01 — Hierarchy
- **Decision:** Parent/child locations, and how deep?
- **Timing:** **Before implementation** of Location (minimal: allow optional parent or not).

### OQ-LOC-02 — Location types
- **Decision:** The initial type list and whether it is governed.
- **Timing:** **Before implementation** (minimal list).

### OQ-LOC-03 — Relationship to Party
- **Resolved — option A2: no Party domain for now.** Every holder of stock is a location: the company's own places, and **customers, dealers and suppliers**. An outside location carries a `holder_type` (customer | dealer | supplier | …) and a **polymorphic link** to its holder's record, held as an external identifier of the location (`OQ-LOC-06`, `OQ-LOC-07`).
- **Why polymorphic:** kinds of holder will keep growing and splitting. This is a second-hand business; anyone, e.g. a person on Facebook, can become a dealer. A new kind of holder is a new `holder_type`, not a schema change.
- **The division of work:** Inventory holds the **quantity**; the location is a standard label answering **who** (holder), **what kind** (`type`) and **where** (`name`, and address details if added).

### OQ-LOC-07 — Where the holder records live — **RESOLVED (for now)**
- **Decision:** ~~Which system owns the customer, dealer and supplier records, and what does the location keep about the holder?~~
- **Answer:** the records **stay in the system that owns them**. The location stores `holder_type`, the record's id **as an external identifier of the location** (e.g. `seller_app/customer_id = c_881`, which also carries the owning system as `source`), and — **when the registration provides them** — a `holder_name_snapshot` and an `address_snapshot`. A holder known in several systems is simply several identifiers on one location. The same approach as `user_name_snapshot` for users (`OQ-AUTHZ-02`).
- **Consequence:** Location never depends on another system's tables to say who holds stock, and holds only the personal details a registration chose to send.

### OQ-LOC-08 — Which location types count as "we have it" — **RESOLVED (mechanism)**
- **Answer:** **the location type decides it, through a fixed mapping.** Each location type maps to one of:
  - **ours, on our premises** — e.g. warehouse, storage area;
  - **ours, held by someone else** — e.g. a dealer holding our piece on consignment, a supplier restoring it. It needs logistics: someone picks it up or drops it off;
  - **no longer ours** — e.g. a customer.
- "Do we have it?" is the quantity at locations in the first two groups; "is it with us?" is the first group only.
- **Condition this relies on (confirmed, `INV-INV-13`):** Inventory records only stock the company **owns or has sold**. Stock at a dealer location is then always *ours*. A piece a dealer owns and we merely want is **not** inventory — that is the wish list (`OQ-ITEM-13`).
- **Still open (small):** the list of location types and their mapping (`OQ-LOC-02`).
- **Timing:** **Before implementation** of Location if such locations are in phase 1; otherwise deferrable.

### OQ-LOC-04 — Logical locations and stock-capable types
- **Decision:** Are "in transit", "sold", "lost" locations, or do movements use null from/to? Which types can hold stock?
- **Timing:** Coupled to OQ-INV-03; **before implementation** of Inventory.

### OQ-LOC-05 — Governance
- **Decision:** Who creates locations, via which application; naming conventions to avoid duplicates.
- **Timing:** **Before implementation** (small).

### OQ-LOC-06 — Location identifiers — **RESOLVED**
- **Decision:** ~~Do locations need external identifiers (barcodes, legacy IDs) using the same pattern as items?~~
- **Answer:** **yes, the same pattern as items**: `LocationIdentifier` records (type from a governed `LocationIdentifierType` table that users can extend, value stored exactly as given, `source`).
- **What it is for:** an external app that creates a location stores its own id on it. Other apps can **find that existing location and register their own id** on it, instead of creating a duplicate. A Scanner shelf barcode is another identifier of the same kind.
- **One value → one location** (unlike items, `OQ-ID-01`): the purpose is to find *the* existing location, so a lookup must give one answer.
- **Find before create:** an app looks the location up by its own id, then by name / address, and only then creates it — with its own id attached. This is the guard against duplicates (`OQ-LOC-05`).

### OQ-PARTY-01 — Is Party needed in phase 1? — **RESOLVED**
- **Answer:** **yes — for Inventory, not for Item.** Inventory must eventually answer: *do we have this item? If yes, where? If not, is it with a dealer or a supplier (so we can arrange delivery), or at a customer (sold)?*
  - **Locations** are places the company owns;
  - **Parties** are entities outside that ownership — dealers, suppliers, customers — that the company has to coordinate with, or whose holdings or consumption it has to understand.
- **Item → Party:** not needed now; the item holds no supplier reference.
- **Superseded by `OQ-LOC-03`:** the need is real, but it is met by **locations** — customers, dealers and suppliers are locations with a polymorphic link to their record. **No Party domain is built for now**; `OQ-PARTY-02` … `05` are dormant.

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

### OQ-API-02 — Command granularity — **RESOLVED**
- **Decision:** ~~Fine-grained intent commands, coarse `UpdateItem`, or both.~~
- **Answer:** **Both, built in two layers.**
  - **First, atomic services**, each with a single responsibility: set dimensions, set weight, set `sku`, set a property, add an image, reorder images, …
  - **Then an orchestrating `UpdateItem`** that accepts one big update (e.g. a whole form) and calls those atomic services in **one transaction**, with one version bump.
- **Consequence:** the rules for each part live in exactly one place — the atomic service — and the big update only groups them. Events follow the same shape (`OQ-EVT-01`).

### OQ-API-03 — Bulk / batch operations
- **Decision:** Needed for migration and imports? Same validation and per-item outcomes?
- **Timing:** **Before implementation** if migration uses the API (it should).

### OQ-API-04 — Query and search capabilities — **RESOLVED (minimum)**
- **Answer — the minimum set:** filter by item type, category, property value (by property name across all types when no type is given — `OQ-PROP-01`), identifier value or prefix, and deleted-or-not; with paging and sorting.
- **Later:** free-text search.

### OQ-API-05 — Direct read access to domain databases
- **Decision:** Prohibited for applications (proposed); what about reporting/analytics?
- **Timing:** **Before implementation.**

### OQ-API-06 — Error contract
- **Decision:** How validation errors, version conflicts, identifier conflicts, authorization errors and idempotent replays are represented.
- **Timing:** Design session; **before implementation**.

## Events

### OQ-EVT-01 — Final event taxonomy — **RESOLVED (item events)**
- **Answer:** **one event per atomic command** (`ItemDimensionsChanged`, `ItemSkuChanged`, `ItemImageAdded`, …).
- **The orchestrator silences them.** When the orchestrating `UpdateItem` runs several atomic services (`OQ-API-02`), their individual events are **not** broadcast; the orchestrator broadcasts **one** event for the whole update, naming the parts that changed.
- **Still open:** the event names for reference data (type renamed, type deleted with items moved, definition changed) and for the other domains.

### OQ-EVT-02 — Payload shape — **RESOLVED**
- **Answer:** **both.** The event's name (and, for the orchestrator's event, its list of changed parts) says *what* changed; its body carries the **full item after the change**. A consumer that missed an event simply overwrites its copy with the latest one.

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

### OQ-AUTHZ-01 — Which application may issue which commands — **RESOLVED (for now)**
- **Decision:** ~~A per-application command allowlist (e.g. Scanner: `transfer` only; Manager: everything).~~
- **Answer:** **No per-application permissions yet.** Every registered application may issue every command and query. All applications are owned by the business, so none is restricted today.
- **Authentication: API keys.** An application registers and receives an **API key**, which it sends with every request. Requests with an unknown, expired or revoked key are rejected. Tables, following the industry standard:

  ```
  Source            ← the registered application / system; one row per source (OQ-ID-05)
      id
      name              ← unique; supplied in the registration request, e.g. shopify-store-x
      created_at

  ApiKey
      id
      source_id
      key_prefix        ← the first few characters, shown in listings so a key can be recognised
      key_hash          ← only a hash is stored; the full key is shown once, at creation
      created_at
      last_used_at
      expires_at        ← optional
      revoked_at        ← set on revoke; the row is kept
      scopes            ← empty for now; where permissions go later
  ```

  - **Rotation:** a source may have several active keys, so a new key can be issued before the old one is revoked, without downtime.
  - **Across domains (proposed):** keys are issued and checked centrally, so one application registration and key work with every domain.
- **Later:** a permission layer will be added on top of this authentication layer. The API should not assume "all apps can do everything" anywhere that would make that addition hard.

### OQ-AUTHZ-02 — Actor identity — **RESOLVED**
- **Decision:** ~~Do commands carry only the application identity, or also the end user?~~
- **Answer:** **both.** A request carries the **application** (known from its API key) and the **user's name as a snapshot** (`user_name_snapshot`), and both are recorded — e.g. in the polymorphic history table.
- **Not a user id:** storing external users' ids would make the Item Domain responsible for other systems' users, which it is not. A name snapshot records *who* without depending on another system's user table.
- **The Item Domain's own users** are people who access the domain directly to make changes without going through an external app; those users belong to the Item Domain.
- **Consequence:** the domain cannot verify a name sent by an external app; it records it as sent, next to the app that sent it.
- **Still open (small):** for the domain's own direct users, whether their id is recorded as well as the name.

### OQ-AUTHZ-03 — Reference-data administration rights — **PARTIALLY RESOLVED**
- **Resolved:** classification vocabulary (categories, item types, property definitions, external identifier types) — **anyone**, recorded in the polymorphic history table (`OQ-CLS-02`).
- **Still open:** location types and movement types.
- **Timing:** **Before implementation.**

## Migration

### OQ-MIG-01 — System of record for bootstrap
- **Decision:** Which application's item data seeds the Item Domain when several hold the same item with different values?
- **Timing:** **Before implementation** of migration.

### OQ-MIG-02 — Duplicate detection (no merge available)
- **Decision:** How is the same physical thing, recorded in several applications, recognised **before** items are created — shared external identifiers, manual review?
- **Why it matters:** There is no merge operation (`OQ-ITEM-04` resolved), so duplicates cannot be repaired afterwards by fusing records. The only remedy is to soft-delete one — and because uniqueness is permanent (`INV-CID-06`), the deleted record's `article_number` and `sku` stay burned forever. **Duplicate detection therefore has to happen up front, during migration, rather than as cleanup afterwards.**
- **Affects:** Migration tooling, identifier model, Manager app.
- **Timing:** **Before implementation** of migration. Raised in priority by the no-merge decision.
### OQ-MIG-03 — Cut-over strategy
- **Decision:** Dual-write period? Strangler pattern per application? Who is authoritative during transition?
- **Timing:** **Before implementation** of migration.

### OQ-MIG-04 — Shopify's role — **RESOLVED**
- **Decision:** ~~Is Shopify a *source* of item facts, a *consumer*, or both — and if both, which wins on conflict?~~
- **Answer:** **Both — and the same holds for every connected application.** Shopify, like all the other apps connected so far, both **supplies** item facts to the Item Domain and **consumes** them.
- **Consequence:** no application is the authority over another; the **Item Domain** is the only authority. Facts from Shopify enter through the same commands as facts from any other app, and conflicting writes are caught by the ordinary version check (`expected_version`) — the second writer gets a version conflict and must re-read, whichever app it is.

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
| 1 | ~~OQ-ITEM-03 (units)~~ | **Resolved:** mm and g, unit in the field name, conversion in clients. |
| 2 | ~~OQ-ID-01 / 02 / 03~~ | **Resolved:** a value may sit on several items; resolution returns a list. |
| 3 | ~~OQ-CID-03 / 04 / 05~~ | **Resolved:** strings with no format rule; mutable, with former values kept and still matched on lookup; lowercase, spaces → `-`. |
| 4 | ~~OQ-PROP-03~~ | **Resolved:** changes allowed; non-conforming items listed and resolved through dedicated endpoints. |
| 5 | ~~OQ-CLS-02~~ | **Resolved:** anyone may change reference data; every change recorded in the polymorphic history table. |
| 6 | ~~OQ-API-02 / OQ-EVT-01 / OQ-EVT-02~~ | **Resolved:** atomic services + orchestrating `UpdateItem`; one event per command, silenced under the orchestrator's single event; events carry the full item. |
| 7 | ~~OQ-AUTHZ-01 / 02~~ | **Resolved:** API keys, no per-app permissions yet; requests carry the app and `user_name_snapshot`. |
| 8 | OQ-MIG-01 / 02 / 04 Seeding, duplicates, Shopify role | The domain is empty without migration, and duplicates must be caught **up front** — there is no merge. |
| 9 | OQ-INV-02 / 03 Negative stock, operation semantics | Inventory cannot be built on candidate operations. |
| 10 | ~~OQ-IMG-01~~ | **Resolved:** `position` column; assigned from list index when not supplied. |
| 11 | OQ-IMG-04 (dead URLs) | Model resolved (absolute URL + optional storage type/key) and `OQ-IMG-05` resolved (three fields). Still open: detecting and repairing a dead URL. |
