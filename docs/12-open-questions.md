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
### OQ-ITEM-03 — Dimensions and weight — **RESOLVED (universality) / OPEN (units)**
- **Decision:** ~~Are dimensions and weight universal first-class attributes or classification-driven properties?~~
- **Answer:** **Universal first-class canonical attributes.** Every physical object has dimensions and a weight regardless of what kind of thing it is, so they belong to the item itself.
- **The rule this establishes:** *universal physical facts are first-class attributes; type-conditional facts are properties.* Properties are conditional attributes of an item **given its category and type** — a seat count means nothing for a lamp. This is the test for any future candidate field.
- **Universal does not mean mandatory.** They are nullable at creation and filled in when the Worker measures the piece after intake (`OQ-ITEM-06`). "Universal" describes what kind of fact it is, not when it must be present.
- **STILL OPEN — units.** Nothing specifies the unit for each dimension or for weight, nor whether the unit is fixed domain-wide or stored per value. Unit ambiguity corrupts data silently: a `220` that might be centimetres or millimetres is worse than no value at all.
- **Timing:** units **before implementation** — this is a schema decision that cannot be retrofitted cheaply once data exists.
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
- **Answer:** `article_number`, `category`, `item_type`, and all **required properties** for that classification. **Nothing else is mandatory.**
- **Consequence for `OQ-PROP-03`:** required properties are enforced **at creation**, which settles the "when" half of that question — and *sharpens* the other half rather than softening it. An item created legitimately before a property definition became `required` would be retroactively invalid, and nothing yet says what happens then.
- **Consequence for dimensions and weight:** universal canonical attributes (`OQ-ITEM-03`) but **not** mandatory at creation — the Worker measures the piece after intake. They are nullable at creation and filled in later, the same shape as `sku`. Do not make them `NOT NULL`.
- **Note:** `category` and `item_type` are both required as input even though item type determines category, which bears on whether `item.category_id` should be stored redundantly at all (`OQ-CLS-05`).
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
- **Answer:** **Canonical.** An item's images are part of its canonical description — the visual answer to "what is this thing?". Modelled as `ItemImage` link records holding an absolute `url` and a storage/source `type`. The Item Domain owns the references; the image files stay in application or third-party storage.
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
  - **restore is possible**: clearing the marker undeletes the item;
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

- **Consequences:** `article_number` is required at creation; `sku` is absent until publication (`OQ-CID-02` resolved, `INV-CID-07`). Lookups and projections must tolerate items with no `sku`. **A missing `sku` must not be read as listing status** — the Item Domain has no concept of publishing and must never gain one; listing state stays with the Seller application (`INV-ITEM-02`).
### OQ-CID-02 — Are `article_number` and `sku` mandatory? — **RESOLVED**
- **Decision:** ~~Must both be present at creation? Can an item have one but not the other?~~
- **Answer:** **`article_number` yes, `sku` no.** A piece is registered with an article number; the SKU is assigned later, when the restored piece is published for sale. An item therefore exists in a perfectly valid state with no `sku`.
- **Consequences:** `sku` is nullable, and every query, projection and integration must handle its absence. Uniqueness on `sku` is enforced only across items that have one. Note this does **not** reopen stub creation — classification (`OQ-ITEM-06`) and `article_number` are both still required at creation.
### OQ-CID-03 — Format and validation rules
- **Decision:** What formats are legal for each (`A-4932`-style patterns, length, allowed characters)? Who defines and maintains the scheme?
- **Why it matters:** The domain is responsible for *enforcing* format on supplied values. Until the rules exist it can enforce uniqueness but nothing else, and malformed numbers enter permanently.
- **Affects:** Validation, Manager/intake UI, migration.
- **Timing:** **Before implementation.**

### OQ-CID-04 — Are they mutable?
- **Decision:** May an `article_number` or `sku` be corrected or reassigned after the fact? If so, is the old value retained, and can it be reused?
- **Why it matters:** Typos in supplied values are inevitable. Mutability is *safe* for application foreign keys because `item_id` shields them — but it breaks anything printed on a physical label, and it interacts with uniqueness (`OQ-CID-05`).
- **Affects:** Commands, events, labels/printing, audit.
- **Timing:** **Before implementation.**

### OQ-CID-05 — Uniqueness scope and normalization — **PARTIALLY RESOLVED**
- **Resolved:** uniqueness is **permanent and global**. A soft-deleted item keeps its `article_number` and `sku`, and those values are never reused by another item (`OQ-ITEM-04`).
- **Still open:** value normalization. Is matching case-sensitive? Is whitespace trimmed? Are leading zeros significant?
- **Why it matters:** Uniqueness cannot be enforced reliably without a normalization rule — `"A-4932"`, `"a-4932"` and `" A-4932 "` must either be the same identity or knowingly different. Scanner and keyboard input are both noisy.
- **Affects:** Validation, lookup queries, migration deduplication.
- **Timing:** **Before implementation.**
### OQ-ID-01 — Uniqueness scope of identifiers
- **Decision:** Is `(namespace, identifier_type, value)` unique across all items? Per namespace only? Never enforced? Configurable per namespace/type?
- **Why it matters:** Resolution is only unambiguous if uniqueness holds. Without it `ResolveIdentifier` must return sets, and Scanner/integration flows need a disambiguation step.
- **Affects:** Identifier model, `AddItemIdentifier` validation, `ResolveIdentifier`, migration.
- **Timing:** **Before implementation.**

### OQ-ID-02 — Multiple external identifiers of the same type on one item
- **Decision:** May an item carry two `shopify/variant_id`s (e.g. listed twice), or two legacy IDs from the same system?
- **Why it matters:** Integrations need to know which one is current. (This no longer affects the Worker projection's `article_number`, which is a canonical attribute and therefore singular.)
- **Affects:** Identifier model, Shopify sync.
- **Timing:** **Before implementation.**

### OQ-ID-03 — Same external identifier on multiple items — **PARTIALLY RESOLVED**
- **Resolved — many is possible.** Identity grouping is a business decision (`OQ-ITEM-12`), so the same foreign product-level identifier can legitimately back more than one Item: the business may register part of a supplier delivery under one identity and part under another, and a later identity split produces two items that both trace to the supplier's single article number. The proposed default `INV-ID-04` — one value resolves to at most one item — therefore **cannot be assumed globally** and must be relaxed per namespace.
- **Still open:** *which* namespaces are one-to-one and which are one-to-many. `shopify/variant_id` is probably one-to-one if each published identity gets its own variant; `supplier/article_number` is probably one-to-many. Also open: how the resolution API represents a multi-item answer without silently picking one.
- **Affects:** Identifier model, resolution API, migration matching.
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

### OQ-ID-06 — External identifier lifecycle
- **Decision:** On `RemoveItemIdentifier`, is history retained (so an old Shopify ID can still be traced)? Can an identifier be reassigned from one item to another?
- **Why it matters:** Traceability during and after migration. Reassignment is **not** needed for merging, which does not exist (`OQ-ITEM-04` resolved), but it may still be needed to move a Shopify or legacy identifier off a soft-deleted duplicate onto the surviving item.
- **Affects:** Identifier model, events.
- **Timing:** **Deferrable**, unless duplicate handling during migration needs it.
### OQ-ID-07 — `item_id` format
- **Decision:** UUID, prefixed string (`itm_…`), other? Generated by the domain only?
- **Why it matters:** Appears in every application, label, URL.
- **Affects:** Everything that stores IDs.
- **Timing:** **Before implementation.**

### OQ-ID-08 — Does the Item Domain generate business identifiers? — **RESOLVED**
- **Decision:** ~~Does the domain *issue* article numbers / SKUs, or only *record* them?~~
- **Answer:** **No.** Values for `article_number` and `sku` are **supplied** by the creating application or person. The Item Domain enforces uniqueness and format but does not generate them.
- **Consequence:** The domain owns no numbering sequence, but it does need format rules to validate against (`OQ-CID-03`), and it must handle the case of an item created before its numbers are known (`OQ-CID-02`).

### OQ-ID-09 — External value normalization
- **Decision:** Case sensitivity, whitespace, leading zeros, per namespace/type.
- **Why it matters:** Uniqueness cannot be enforced reliably without it; scanner input is noisy. (The equivalent question for our own identifiers is `OQ-CID-05`.)
- **Affects:** Identifier validation and resolution.
- **Timing:** **Before implementation.**

### OQ-ID-10 — Confirm that supplier article numbers are external
- **Decision:** Confirm that a *supplier's* article number is an external identifier, not canonical identity.
- **Why it matters:** This is **inferred**, not stated. It follows from the issuing-authority principle — the supplier assigns the value, so we cannot guarantee its uniqueness or stability — and from your correction naming only *our* article number and SKU as canonical. But it is close enough in wording to `article_number` that it should be confirmed rather than assumed.
- **Affects:** Identifier model, Shopify/supplier integrations, migration.
- **Timing:** **Before implementation** (small, but it decides where supplier numbers are stored).

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
- **Decision:** With `name` removed from the item (`OQ-ITEM-02` resolved), is there *any* canonical descriptive text — say a `description` or `model_name` property on the item types that warrant one — or is all descriptive text channel copy owned by Seller/Shopify?
- **Why it matters:** The default is now "none", which is clean, but it means a Shopify listing title is composed entirely by the integration. If some neutral description should be shared across channels, it needs a property definition; otherwise each channel writes its own and they drift.
- **Affects:** Property vocabulary, Seller, Shopify integration.
- **Timing:** **Before implementation** for the initial property set.
### OQ-IMG-01 — Ordering and primary image
- **Decision:** How does a consumer know which image to show when it can only show one, and in what order the rest appear? An explicit `is_primary` flag, an integer `position`, or "oldest first"?
- **Why it matters:** Every list view, card and Shopify listing needs "the" image. With no rule in the canonical model, each application picks its own — which is precisely the divergence the Item Domain exists to prevent.
- **Affects:** `ItemImage` model, every application projection, Shopify sync.
- **Timing:** Consciously deferred. **Forced as soon as any application displays a single image for an item**, which in practice means the first list view built.

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

### OQ-IMG-04 — URL durability and storage ownership
- **Decision:** Does the Item Domain validate that a URL resolves? What happens when an application's storage is retired or a CDN path rotates — who repairs the canonical reference? Is image storage expected to outlive the application that uploaded it?
- **Why it matters:** This is the **one place where canonical state depends on infrastructure no domain controls**. A dead URL is canonical data that is silently wrong, and nothing in the current model detects it. It also determines whether storing a bare absolute URL is sufficient or whether the model needs storage-type + key so references can be rebuilt.
- **Affects:** `ItemImage` model, every uploading application, operations/monitoring.
- **Timing:** **Before implementation** — at least a policy, because the model choice depends on it.

### OQ-IMG-05 — Semantics of `type`
- **Decision:** Is `type` the application that uploaded the image, or the storage backend the file lives in? Today these coincide; they can diverge (two apps writing to one bucket, or one app migrating storage). Is the set of values a governed list, as proposed for identifier namespaces (`OQ-ID-04`)?
- **Why it matters:** Decides whether `type` is merely provenance or can actually be used to re-resolve and migrate URLs (`OQ-IMG-04`).
- **Affects:** `ItemImage` model, validation, migration.
- **Timing:** **Before implementation** (small, but it changes what the field means).

### OQ-IMG-06 — Aggregate membership and uniqueness
- **Decision:** Do image changes increment `item.version` (see `OQ-ITEM-05`)? May the same URL be attached to more than one item? Is there a minimum (must an item have at least one image?) or maximum count?
- **Why it matters:** Version membership is **resolved** — image changes increment `item.version` like any other canonical change (`OQ-ITEM-05`), so a Worker uploading photos does conflict with concurrent edits, deliberately. What remains is whether the same URL may be attached to more than one item, and whether there is a minimum or maximum count.
- **Affects:** Command semantics, events, projections.
- **Timing:** **Before implementation.**

### OQ-IMG-07 — Category / ItemType presentational assets
- **Decision:** Are `icon_url` and `image_url` mandatory on every category and item type? What does an application render when one is missing? Who supplies them, through which application (see `OQ-CLS-02`)?
- **Why it matters:** Reference data is rendered everywhere. A missing icon needs a defined fallback, or every application invents its own — reintroducing the divergence the shared vocabulary was meant to remove.
- **Affects:** Reference-data commands, every application UI.
- **Timing:** **Before implementation** (small).

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

### OQ-INV-08 — Party references and ownership
- **Decision:** Do movements record a `party_id` (supplier on receive, customer on sell)? Is stock ownership/consignment a concept?
- **Timing:** **Deferrable**; tied to OQ-PARTY-01.

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

### OQ-MIG-02 — Duplicate detection (no merge available)
- **Decision:** How is the same physical thing, recorded in several applications, recognised **before** items are created — shared identifiers, matching on supplier article number, manual review?
- **Why it matters:** There is no merge operation (`OQ-ITEM-04` resolved), so duplicates cannot be repaired afterwards by fusing records. The only remedy is to soft-delete one — and because uniqueness is permanent (`INV-CID-06`), the deleted record's `article_number` and `sku` stay burned forever. **Duplicate detection therefore has to happen up front, during migration, rather than as cleanup afterwards.**
- **Affects:** Migration tooling, identifier model, Manager app.
- **Timing:** **Before implementation** of migration. Raised in priority by the no-merge decision.
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
| 1 | OQ-ITEM-03 (units) Units for dimensions and weight | A `220` that might be cm or mm corrupts data silently, and units cannot be retrofitted cheaply. |
| 2 | OQ-ID-01 / 02 Which namespaces are one-to-one vs one-to-many | `OQ-ID-03` settled that many *is* possible; the per-namespace policy and how resolution returns several items are not. |
| 3 | OQ-CID-03 / 04 / 05 Business identifier format, mutability, normalization | The domain enforces rules on supplied values that it has not yet been given. |
| 4 | OQ-PROP-03 Definition evolution | Sharpened: required properties are now enforced at creation, so an item created before a property became required is retroactively invalid. |
| 5 | OQ-CLS-02 Reference-data governance | Otherwise vocabulary changes silently break items. |
| 6 | OQ-API-02 / OQ-EVT-01 / OQ-EVT-02 Command granularity, event taxonomy, payload shape | Contracts other teams build against — and granularity now drives how often version conflicts occur. |
| 7 | OQ-AUTHZ-01 / 02 Who may do what, as whom | Cannot build the write boundary without it. |
| 8 | OQ-MIG-01 / 02 / 04 Seeding, duplicates, Shopify role | The domain is empty without migration, and duplicates must be caught **up front** — there is no merge. |
| 9 | OQ-INV-02 / 03 Negative stock, operation semantics | Inventory cannot be built on candidate operations. |
| 10 | OQ-IMG-01 Primary image | Promoted: humans now identify a piece partly *by its picture*, so "which one shows" is load-bearing. |
| 11 | OQ-IMG-04 / 05 Image URL durability and `type` semantics | Canonical state depends on storage no domain owns; the model choice depends on the answer. |
