# 00 — Owner Guide (plain language)

**Audience:** The business owner. No technical background needed.
**Purpose:** Explain what this architecture is, why it is being built, what has already been decided, and which business decisions are still needed. The technical documents (`docs/README.md` and 01–12) remain the source of truth; this guide translates them.

---

## 1. The problem in one paragraph

Today each of your applications (Manager, Worker, Seller, Scanner, and the Shopify connection) keeps **its own private copy** of what an item is. The same sofa can exist four times, with four different IDs and slightly different information in each — one app says 220 cm wide, another says 218. Nobody can say which one is correct, and connecting data between apps is manual and fragile.

## 2. The solution in one paragraph

Create **one central "source of truth"** for shared business information. Every item gets **one permanent ID** that all apps use. Apps still do their own jobs (photographing, listing, scanning), but when they need to know *what an item is*, *how many we have and where*, or *which place we mean*, they ask the central source instead of keeping their own version.

**Analogy:** moving from every department keeping its own Excel file of products to one shared master catalogue. Departments still keep their own to-do lists — but the product facts live in one place, and only that place is allowed to change them.

---

## 3. The central "domains"

A *domain* is one area of responsibility with a single owner of the truth.

| Domain | The question it answers | Example | Status |
|---|---|---|---|
| **Item** | What is this thing? | "Article 1042 is a leather sofa, 2200 × 900 × 850 mm, with these photos." | Main focus; most decisions made |
| **Inventory** | How many do we have, and where? | "1 unit in Warehouse A; 1 unit at dealer X on consignment." | Early design; several rules still open |
| **Location** | Which place do we mean? | "Warehouse A", "Dealer X", "Customer Y" | Minimal |
| **Party** (people/companies) | Which person or company? | — | **Not built for now.** Customers, dealers and suppliers are handled as *locations* instead. |

They are kept separate on purpose, so a change in how stock is counted doesn't risk breaking how items are described.

## 4. The golden rule

> **Central domains own facts that are true for every app. Apps own their own work-in-progress.**

| Belongs centrally (a fact) | Stays in the app (work in progress) |
|---|---|
| Type, dimensions, weight, properties, photos | "Waiting for photography" |
| Article number, SKU, Shopify ID | "Selected for this week's campaign" |
| How many at which location | "Listing is still a draft" |

A sofa being "awaiting photography" is not a fact about the sofa — it's a step in the Worker app's process. Keeping these apart keeps the central catalogue stable while apps and processes change.

---

## 5. What has been decided — in plain terms

**What an "item" is**
- An item is **the identity the business gives to a piece** — usually one unique vintage piece, but it can also cover several identical units (e.g. 60 identical chairs as one item). *How many* units exist is only recorded in Inventory.
- Whether two physical things share one item or get separate ones is **your team's decision at registration**. The system doesn't guess.
- Items are **not linked to each other** (no "variants", "sets" or "parent products").

**How an item is identified**
- Every item gets a permanent system ID (like `itm_…`) that apps use behind the scenes.
- **Article number** — given when a piece is registered at intake. Required. Means "we have it and are tracking it".
- **SKU** — usually given when the restored piece is published for sale. Always optional, and can be removed again. A missing SKU does **not** mean "not for sale" — listing status lives in the Seller app.
- Both are typed in by the app or person (the system doesn't generate them), are **unique forever**, and **can be corrected**. The old number keeps working, so an old printed label still finds the piece.
- Shopify IDs, old-app IDs and similar are attached as extra "tags". Several items may share such a tag.

**What an item contains**
- A **type** (e.g. Sofa), which decides its category (e.g. Furniture).
- **Dimensions in millimetres and weight in grams**, each optional.
- **Properties** that depend on the type (material, seat count, bulb type…).
- **Photos** (links to where the images are stored).
- **No official name or description.** People recognise a piece by its **type, photo and article number/SKU**. Listing titles and texts belong to the sales channel (Seller/Shopify).

**Changes and history**
- Items can be **deleted, but only "soft-deleted"** — hidden, not destroyed, and restorable. There is **no merging** of items.
- **Anyone may change categories, types and properties.** Control comes from a complete **change history** (what, when, who), not from permissions. If a change makes existing items incomplete, those items appear on a **"needs fixing" list** instead of being blocked.
- Every change records **which app** made it and **the person's name**.
- For now **every app may do everything** — apps identify themselves with an API key (a password for apps).

**Stock and places**
- Stock changes are recorded as **movements with a reason** (received, moved, sold…), never silently overwritten.
- Stock at a **dealer, supplier/restorer or customer** is stock at a *location of that kind*. A sale moves the piece to the **customer's location**.
- The **type of location** decides whether a piece counts as "ours, here", "ours, held by someone else" or "no longer ours".
- Inventory only records pieces **you own or have sold**. Pieces you want but don't own go in a future **wish list** app.

**Shopify**
- Shopify, like every other app, both **sends** item information in and **receives** it. None of the apps is in charge — only the central Item domain is. If two apps change the same item at the same moment, the second one is told to reload and retry — as long as the app says which version it edited (whether that is mandatory is still open, OQ-CC-01).

---

## 6. Where the project stands

**Blueprint stage.** The concepts and most Item rules are decided. Not yet designed: the technical specifications (API, database structure), the technology and hosting choice, and **the plan for moving existing data into the new system**.

---

## 7. Decisions still needed from the business

Most remaining open questions (doc 12) are technical. These are the ones where **your business knowledge is needed**:

**Needed before building**

| # | Business question | Why it matters | Ref |
|---|---|---|---|
| 1 | **Which app has the most correct item data today?** When apps disagree about a piece, which one wins during the move? | The new system starts empty and has to be filled from somewhere. | OQ-MIG-01 |
| 2 | **How do we spot the same piece recorded in several apps — before importing?** (Matching Shopify IDs? Manual review?) And if a duplicate slips through, can its Shopify/old-app IDs be moved to the item we keep? | There is no merging. A duplicate can only be deleted afterwards, and its article number is then used up forever. Duplicates must be caught **up front**. | OQ-MIG-02, OQ-ID-06 |
| 3 | **During the switch-over, which system is in charge** — old apps or the new center? All at once, or app by app? | Avoids two "truths" during the transition. | OQ-MIG-03 |
| 4 | **Can stock go negative?** E.g. something sold online before it was registered as received. | Decides whether the system blocks such a sale or records it and flags it. | OQ-INV-02 |
| 5 | **What does each stock movement mean in practice?** E.g. the difference between "received" and "put on a shelf"; what "returned" does; are "in transit" or "lost" places of their own? | Stock counting can't be built on guesses. | OQ-INV-03, OQ-LOC-04 |
| 6 | **What kinds of locations do we have, and which group does each belong to?** The three groups are decided (ours-here / ours-held-elsewhere / no longer ours); the list of types is not. Can locations sit inside others (warehouse → shelf)? What happens to stock at a location that closes? | Drives "do we have it / where is it" answers. | OQ-LOC-01/02/08, doc 05 |
| 7 | **Who creates new locations, and what naming rules do we use?** (The "look it up before creating it" rule is already decided.) | Duplicate locations split stock across two records. | OQ-LOC-05, OQ-LOC-06 |
| 8 | **Who creates new items?** Only Manager? Worker at intake? Shopify import? | Affects the daily intake process. | OQ-APP-04 |
| 9 | **Does the Scanner need to work offline** (e.g. in a warehouse with bad Wi-Fi)? | Changes how the Scanner is built. | OQ-APP-03 |
| 10 | **What must we be able to trace, and for how long?** (Who changed what, when, why.) | Sets the audit and retention requirements. | OQ-OPS-01 |
| 11 | **What should apps do if the central system is briefly down?** Keep working read-only? Queue changes? | Defines acceptable downtime behaviour. | OQ-OPS-02 |
| 12 | **Who may manage location types and movement types?** (Categories/types are already "anyone".) | Last part of the permissions picture. | OQ-AUTHZ-03 |
| 13 | **What personal data may we keep about customers and dealers** (name and address stored on their location), and for how long? | Privacy/GDPR. Not yet covered by any open question in doc 12. | OQ-LOC-07, OQ-PARTY-04 |

**Can wait (but must not be improvised later)**

| # | Business question | Ref |
|---|---|---|
| 14 | Do we need to track "reserved for a customer", "damaged", "display model"? | OQ-INV-04/05 |
| 15 | Should photos say what they show (front, detail, damage) and carry alt text for Shopify? | OQ-IMG-02/03 |
| 16 | Do we need category and property names in several languages? | OQ-PROP-05 |
| 17 | Who repairs a photo link that stops working? | OQ-IMG-04 |
| 18 | When an item is deleted while stock still exists, who cleans up that stock? | OQ-ITEM-10 |
| 19 | Do we need sub-categories (more than Category → Type)? | OQ-CLS-01 |
| 20 | Grouping similar pieces into a "catalogue" — deliberately postponed until the business asks for it. | OQ-ITEM-11 |

*Not on this list on purpose:* splitting or combining item identities later. It is decided that this is future work and must **not** be designed now (OQ-ITEM-07, INV-ITEM-21).

---

## 8. Glossary

| Term in the docs | Plain meaning |
|---|---|
| **Domain / canonical domain** | A central area of responsibility that owns the official truth for one kind of information. |
| **Canonical** | Official, authoritative, the one true version. |
| **item_id** | The permanent system ID of an item (`itm_…`). Used by apps behind the scenes; never reused. |
| **article_number** | Your own number for a piece, given at registration. Required, unique forever, correctable. |
| **sku** | Your own sales number, given when a piece is published for sale. Optional until then. |
| **External identifier** | A code from another system (Shopify ID, old-app ID) attached to an item as a tag. |
| **Category / Item type** | Classification, e.g. Furniture → Sofa. The type decides which properties are allowed. |
| **Property** | An attribute like material, seat count, bulb type. Defined per type. |
| **Resolution list** | The "needs fixing" list of items that no longer match their type's rules after a rule change. |
| **Soft delete** | Hidden rather than destroyed; can be restored. Its numbers are never reused. |
| **History table** | The central log of every change: what, when, which app, which person. |
| **API key** | A secret key each app uses to identify itself to the central system. |
| **Inventory position** | "How many of item X at location Y." |
| **Movement** | A recorded stock change with a reason: received, moved, sold, returned, adjusted. |
| **Holder type** | For outside locations: whether it's a customer, dealer, supplier, etc. |
| **Workflow state** | An app's own progress status (e.g. "awaiting photography"). Never stored centrally. |
| **Command / Query / Event** | A request to change something / a request to read something / an announcement that something changed, so other apps can update. |
| **Projection** | An app's local copy of central data for fast display. A cache, never the truth. |
| **Invariant** | A rule that must always hold (e.g. "an article number is never reused"). |
| **Version / concurrency** | Protection against two people or apps overwriting each other's changes. |
| **Migration** | Moving existing data from the current apps into the new central system. |
| **CONFIRMED / PROPOSED / OPEN / RESOLVED / PARTIALLY RESOLVED / WITHDRAWN** | Decided / suggested starting point / not yet decided / answered / answered in part / was decided then reversed — don't bring it back. |
| **INV-… / OQ-…** | Reference numbers: INV = a rule (invariant), OQ = an open (or resolved) question. |

---

## 9. Good questions to ask Claude about this repo

- "Explain the Item domain like I'm a store manager."
- "What happens, step by step, when a worker measures a sofa?"
- "Which decisions are blocking the start of development?"
- "Why is there no merging, and what does that mean for moving our old data?"
- "What happens to existing items if we add a new required property to lamps?"
- "What are the risks of letting anyone change categories?"
- "Propose an improvement to the inventory design and explain the trade-offs."
