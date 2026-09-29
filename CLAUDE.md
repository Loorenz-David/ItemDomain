# CLAUDE.md — context for Claude

## What this repository is
An **architecture documentation package** (no code yet) for a set of centralized business domains — **Item, Inventory, Location** (Party deferred) — shared by the Manager, Worker, Seller, Scanner and Shopify/integration applications. Status: blueprint stage; most Item Domain decisions are made. Not designed yet: concrete API and payloads, database schema, event transport/schemas (item event shape is decided), per-app permissions (API-key authentication is decided), technology/deployment and the migration plan.

## Where things are
- `docs/00-owner-guide.md` — plain-language summary for the business owner. Start here for non-technical questions.
- `docs/README.md` — technical entry point and documentation map.
- `docs/01`–`10` — one topic each (item, identifiers, classification, inventory, location, party, boundaries, commands/events, versioning, app integration).
- `docs/11-invariants.md` — all rules (`INV-*`), each CONFIRMED / PROPOSED / OPEN.
- `docs/12-open-questions.md` — every decision (`OQ-*`) with timing. Its header text ("Nothing here is answered") is outdated: many entries are RESOLVED.

## Rules when answering
1. **Respect the status markers — read the entry body, not just the heading.** CONFIRMED / RESOLVED = decided; a qualified label ("RESOLVED (for now)", "(mechanism)", "(minimum)") is decided only within that scope; a "Still open" line inside a RESOLVED entry is undecided; PROPOSED = starting direction; OPEN = undecided; "Deferred" = intentionally postponed; WITHDRAWN (doc 11) = reversed, never reintroduce. Never present an OPEN item as decided. Cite the `OQ-*` / `INV-*` number when relevant.
2. **Source of truth is `docs/README.md` and docs 01–12.** The owner guide is a translation; if they disagree, the technical docs win — say so. If technical docs disagree with each other, the markers in 11 and 12 win over narrative text in 01–10 and the README — and flag the inconsistency.
3. **Core principle to check every proposal against:** central domains own canonical business facts; applications own workflow state. Never propose putting workflow or listing status (e.g. `awaiting_photography`, `published`) into a domain. A missing SKU does not mean "unpublished" (INV-ITEM-02, OQ-CID-01).
4. **Don't invent** endpoints, schemas, technologies, costs or timelines the docs don't contain. Say "not yet designed" and point to the open question.
5. When proposing changes, state: what changes, why, which docs/invariants/open questions it touches, trade-offs, and whether it contradicts anything CONFIRMED.

## If the reader is the business owner (non-technical)
- Use plain business language and analogies (catalogue, warehouse, shelf, receipt). Avoid jargon; if a term is needed, explain it in one line (see glossary in the owner guide).
- Lead with the answer, then the "why it matters for the business".
- Turn technical open questions into business questions he can answer.
- Proposed changes should be written so they can be forwarded to the developer as-is (a short "For the developer" section with doc references).
