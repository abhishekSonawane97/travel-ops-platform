# PureLuxe Studio — Database Design

**For:** CTO review, before any migration is written
**Date:** 25 July 2026
**Follows:** [01-product-vision.md](01-product-vision.md) v1.0, [02-domain-model.md](02-domain-model.md) v1.0
**Target:** PostgreSQL 15+ on Supabase, with Row Level Security
**Scope:** Schema design only. No migrations, no APIs, no application code.

---

## Reading notes

Tables are numbered **T1–T52** for reference in §9 (migration order). Column types are named in PostgreSQL terms so the design is unambiguous, but nothing here is a migration — no DDL, no ordering guarantees, no `CREATE` statements.

**Every table carries a standard column block**, defined once in §6 and never repeated in §3:

| Block | Columns | Applies to |
|---|---|---|
| **Audit** | `created_at`, `created_by`, `updated_at`, `updated_by` | Every mutable table |
| **Append-only** | `created_at`, `created_by` only | Immutable tables (events, audit, payments, snapshots, access logs) |
| **Soft delete** | `deleted_at`, `deleted_by` | Only tables marked *Soft delete: yes* |
| **Provenance** | `source_review_item_id` | Only tables an intake proposal can create or change |

Where a table's §3 entry omits these, assume the block applies.

---

# 1. Database Design Principles

### 1.1 The schema encodes the domain model, not the other way round

Every table corresponds to a persistent business concept from the Domain Model. There are no tables for derived concepts (§5), no tables invented for the convenience of a screen, and no tables that a consultant would not recognise if the columns were read aloud.

### 1.2 Aggregate boundaries are expressed in foreign key behaviour

This is the single most useful translation of DDD into DDL:

- **Inside an aggregate** — the child cannot exist without the parent. FK is `NOT NULL`, `ON DELETE CASCADE`. Example: `rate_lines → rate_contracts`, `itinerary_components → trips`.
- **Across aggregates** — a reference by identity only. FK is `ON DELETE RESTRICT` (or `NO ACTION`), **never cascade**. Example: `bookings → trips`, `documents → travellers`.

If deleting row A silently deletes row B in another aggregate, the boundary has been violated. Cascades crossing an aggregate boundary should fail review.

### 1.3 UUID primary keys, time-ordered where possible

Every table uses `uuid` primary keys. Prefer **UUIDv7** (`uuidv7()` on PostgreSQL 18; otherwise a `pg_uuidv7`-style function or generation in the application) because it is time-ordered and keeps B-tree inserts and index locality sane on the high-volume tables — events, messages, audit. Fall back to `gen_random_uuid()` where v7 is unavailable.

Rationale for UUID over `bigint`: records originate outside the database (extraction proposals, Sabre mirrors, and one day client-side writes), and UUIDs let identity be assigned before insertion. The cost is index size, which is irrelevant at this volume — low thousands of trips per year.

**No natural keys as primary keys.** Passport numbers, PNR record locators and supplier references all change, get corrected, or turn out to be non-unique. They are attributes with unique constraints where warranted, never identity.

### 1.4 Soft delete is the exception, not the rule

Three classes of table, three policies:

| Class | Policy | Examples |
|---|---|---|
| **Business records with archival meaning** | Soft delete (`deleted_at`) | Families, travellers, suppliers, properties, trips, rate contracts, tasks |
| **Append-only facts** | No delete at all, ever | Events, audit rows, payments, proposals, communications, access logs, extractions |
| **Join/child rows inside an aggregate** | Hard delete, cascading from the parent | Trip party, booking travellers, itinerary days |

Rules that follow:

- **Soft-deleted rows are invisible by default.** RLS policies and every view filter `deleted_at IS NULL`. Nothing relies on application code remembering.
- **Unique constraints on soft-deletable tables are partial indexes** (`WHERE deleted_at IS NULL`), otherwise a deleted row blocks re-creating the same business key.
- **Soft delete never cascades.** Archiving a Family does not archive its Trips — invariant 3 forbids archiving a Family with a live Trip in the first place, and that check belongs in the application, not in a cascade.
- **Documents are the exception in the other direction**: purging is a *hard* delete of the stored file plus a tombstone row (invariant 24). The row survives with its file reference nulled; the bytes do not.

### 1.5 Immutability where the business says immutable

Enforced by trigger (reject `UPDATE`/`DELETE`), not by convention:

`intake.inbound_messages` · `intake.extractions` · `planning.proposals` (once sent) · `money.payments` (once confirmed) · `events.domain_events` · `audit.record_changes` · `documents.document_access_log`

Corrections to immutable records are *new rows*, never edits. A reversed payment is a second payment with a negative amount and a link to the original.

### 1.6 Snapshot strategy — reference for identity, copy for frozen facts

The domain model's most important data rule (invariant 20, "a component's price is copied at selection") produces a consistent pattern:

| Situation | Store |
|---|---|
| Which rate priced this component | `rate_line_id` reference **and** copied `unit_cost`, `currency`, `rate_captured_at` |
| What the client saw in a proposal | Whole-itinerary `jsonb` snapshot, immutable |
| Which traveller a booking names | `traveller_id` reference only — no copy |
| What Sabre reported | Mirror columns, refreshed; never merged into `bookings` |

**Reference when you need the current truth. Copy when you need the historical fact.** A contract superseded next March must never reprice a trip sold last week, and a proposal sent yesterday must render identically forever.

Snapshots are `jsonb` because their shape is historical — a two-year-old proposal must render even after the component model changes. They are never queried for reporting; that is what events are for.

### 1.7 Reference vs duplication

Default to reference. Duplicate only when one of these is true:

1. **The value is a historical fact** (copied rate, proposal snapshot, extraction payload).
2. **The source lives outside the database** (Sabre mirror).
3. **A join would cross an aggregate boundary in a hot path and the value never changes** — currently no case qualifies, and this exception should require justification in review.

Denormalisation for read performance is not on this list. At this volume, a correct join beats a stale copy.

### 1.8 Enumerated states

Lifecycle states from the Domain Model are **native PostgreSQL enum types**, one per lifecycle, named for the concept (`trip_state`, `booking_state`, `review_item_state`). They come from an approved model, they are stable, and the type system catching a typo is worth more than the cost of `ALTER TYPE ... ADD VALUE`.

Two exceptions use lookup tables instead, because they carry attributes and will churn: **loss reasons** and **task templates**.

Free-text status columns are not used anywhere.

### 1.9 Money

`numeric(14,2)` plus a `char(3)` ISO currency, always together, always `NOT NULL` as a pair. Never `float`. Never a bare amount.

**No cross-currency arithmetic in the database.** Trip commercials sum only within a currency; multi-currency rollups are a reporting concern with an explicit rate and date, not a schema concern.

### 1.10 Timestamps

`timestamptz` everywhere, stored UTC. Never `timestamp`.

Travel dates are the exception: a hotel check-in on 14 August is a `date`, not an instant, and forcing it into `timestamptz` invents a timezone the business never stated. **Dates for itinerary and stay boundaries; `timestamptz` for events, deadlines and audit.**

Date ranges use `daterange` where overlap matters (invariant 8, non-overlapping accommodation), so the constraint can be enforced by an exclusion constraint rather than application code.

### 1.11 Versioning

Three different needs, three different mechanisms — conflating them is a common mistake:

| Need | Mechanism |
|---|---|
| What the client saw | `proposals` — immutable snapshot rows, versioned by `version_no` |
| How a record changed over time | `audit.record_changes` — before/after per column |
| Business supersession | Explicit `superseded_by_id` self-reference (rate contracts, invariant 19) |

No table has a generic `version` integer for optimistic locking; contention at this scale does not warrant it. Where two operators genuinely race — accepting the same review item — the review item's state transition is the guard.

### 1.12 Event-ready from day one

`events.domain_events` is an append-only log written in the **same transaction** as the state change that produced it. It serves three purposes at once: the domain event stream, the transactional outbox for future integrations, and the source of the Timeline projection.

This is the cheapest possible version of event-readiness: one table, no broker, no event sourcing. Aggregates keep their current state in their own tables; events record what happened. If a broker is ever needed, it reads from this table.

### 1.13 Correctness over cleverness

Deliberately not used: table inheritance, polymorphic foreign keys, EAV columns, generic "entity" tables, triggers containing business logic, stored procedures as an API layer, partitioning before there is data to partition.

Deliberately used: constraints the database can enforce (composite FKs, exclusion constraints, check constraints), because a small team cannot rely on every code path remembering a rule.

---

# 2. Schemas

Thirteen schemas, aligned to the Domain Model's bounded contexts. The alignment is deliberate: a schema boundary is a visible, greppable reminder of a context boundary, and it makes "which context owns this table?" answerable without a document.

| Schema | Context | Why it exists |
|---|---|---|
| `identity` | Administration | Operator profiles, roles and sensitivity grants. **Separate from Supabase's `auth` schema**, which is managed by the platform and must not be extended. `identity.operators` links to `auth.users` by id and holds everything the business knows about a staff member. |
| `reference` | *(cross-cutting master data)* | Countries, destinations and destination document policies. Small, slow-moving, read by several contexts. Kept out of `crm` and `documents` so neither context appears to own rules that belong to the world. |
| `crm` | CRM | Families, travellers, preferences, contact identities. The core relationship asset. |
| `pipeline` | Enquiry & Pipeline | Enquiries and loss reasons. Separate from `crm` because an enquiry's lifecycle is short and its ownership rules differ. |
| `planning` | Travel Planning | Trips, itineraries, components, proposals. The core design surface. |
| `rates` | Rates & Supply | Suppliers, properties, rate contracts and lines. **A core schema, not master data** — this is one of the two strategic assets, and filing it under `reference` would guarantee it gets built like a lookup table. |
| `booking` | Booking | Bookings and the travellers they name. Separate from `planning` because bookings change on the outside world's schedule. |
| `sabre` | Sabre Sync | The Sabre mirror and sync state. **An anti-corruption layer with a schema boundary to match** — nothing outside this schema stores a record locator or a segment code. |
| `documents` | Documents | Documents and their access log. Isolated because it holds the most sensitive data in the system and gets the strictest RLS. |
| `ops` | Operations | Tasks and task templates. |
| `money` | Trip Money | Payment schedules, milestones, payments. Isolated for auditability. |
| `intake` | Intake | Inbound messages, extractions, review queue, auto-accept policy. **Owns no business record it proposes changes to** — the schema boundary makes the "Intake never writes" rule visible. |
| `conversations` | Conversations | Threads, communications, internal notes. Deliberately separate from `intake`: one asks *what does this mean*, the other *who said what to whom*. This split is the seam WhatsApp plugs into. |
| `events` | *(cross-cutting)* | The append-only domain event log. Owned by no context; written by all. |
| `audit` | Administration | Row-level change history and security-relevant access records. |
| `reporting` | Reporting | **Contains no base tables.** Materialized views and read-only views only. Nothing here is a source of truth; everything is rebuildable. |

### Schema-level rules

1. **Cross-schema foreign keys are permitted but must be by identity only** — never a cascade, never a composite that reaches into another context's internals (with one deliberate exception, §3 `crm.preferences`, noted there).
2. **`reporting` never has anything referencing it.** If a foreign key points into `reporting`, the design is wrong.
3. **`intake` holds no FK from any business table except the nullable `source_review_item_id` provenance pointer.** Provenance points *at* intake; intake does not own business rows.
4. **RLS is enabled on every table in every schema**, including lookup tables. Default deny. There are no exceptions "because it's just reference data."
5. **`events` and `audit` are insert-only for all application roles.**

---

# 3. Tables

## 3.1 `identity`

---

**T1 · `identity.operators`** — a staff member who uses the platform.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `auth_user_id` | uuid | yes | → `auth.users(id)`. Null before first sign-in (invited but not activated). |
| `full_name` | text | no | |
| `email` | citext | no | Work email; also used to attribute inbound mail sent by staff |
| `state` | operator_state | no | `invited`, `active`, `suspended`, `departed` |
| `job_title` | text | yes | |
| `default_assignee` | boolean | no | Default false. Used by task generation. |

- **PK** `id` · **FK** `auth_user_id → auth.users(id)` ON DELETE SET NULL
- **Unique** `auth_user_id` (partial, `WHERE auth_user_id IS NOT NULL`); `lower(email)` where `deleted_at IS NULL`
- **Indexes** `state` (partial `WHERE state = 'active'`)
- **Check** a `departed` operator has `auth_user_id` revoked at the application layer, not nulled — attribution must survive
- **Soft delete** No. Operators reach `departed` and stay. Deleting an operator would orphan every `created_by` in the system.
- **Lifecycle** invited → active → suspended → departed

---

**T2 · `identity.operator_roles`** — role grants. Roles are additive (one person may be Founder *and* Administrator), so this is a set, not a column.

| Column | Type | Null | Notes |
|---|---|---|---|
| `operator_id` | uuid | no | → `identity.operators` |
| `role` | operator_role | no | `consultant`, `operations`, `founder`, `administrator` |
| `granted_at` / `granted_by` | timestamptz / uuid | no / yes | |

- **PK** (`operator_id`, `role`) · **FK** operator ON DELETE CASCADE (inside the aggregate)
- **Indexes** `role`
- **Soft delete** No — revocation is a `DELETE` plus an `audit.record_changes` row.

---

**T3 · `identity.sensitive_grants`** — access to a sensitivity tier, independent of role (domain rule 48).

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `operator_id` | uuid | no | → operators |
| `tier` | sensitivity_tier | no | `identity_documents`, `commercials` |
| `granted_by` | uuid | no | → operators |
| `granted_at` | timestamptz | no | |
| `revoked_at` / `revoked_by` | timestamptz / uuid | yes | |
| `reason` | text | yes | |

- **PK** `id` · **Unique** (`operator_id`, `tier`) partial `WHERE revoked_at IS NULL`
- **Indexes** (`operator_id`, `tier`) partial on active grants — read on every sensitive query, so it must be fast
- **Check** `revoked_at IS NULL OR revoked_at >= granted_at`
- **Soft delete** No — revocation is explicit and part of the record. Administrator role does **not** imply a grant (rule 49).

---

## 3.2 `reference`

---

**T4 · `reference.countries`** — ISO country list. Small, stable, referenced by nationality and destination.

`code` char(2) PK (ISO 3166-1 alpha-2) · `name` text NOT NULL · `alpha3` char(3) · `default_currency` char(3)

- **Indexes** none beyond PK · **Soft delete** No · **Lifecycle** effectively static

---

**T5 · `reference.destinations`** — a place the agency sells: a city, region or island group.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `name` | text | no | |
| `country_code` | char(2) | no | → countries |
| `parent_id` | uuid | yes | Self-reference for region → city |
| `search_vector` | tsvector | no | Generated from name + country |

- **Unique** (`country_code`, `lower(name)`) partial `WHERE deleted_at IS NULL`
- **Indexes** GIN on `search_vector`; `parent_id`
- **Soft delete** Yes

---

**T6 · `reference.destination_document_policies`** — the rules that generate Document Requirements: passport validity windows, visa needs, lead times, by destination and traveller nationality.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `destination_id` | uuid | no | → destinations |
| `nationality_code` | char(2) | yes | Null = applies to all nationalities |
| `document_kind` | document_kind | no | `passport`, `visa`, `insurance`, `id` |
| `passport_validity_months` | smallint | yes | e.g. 6 months beyond return |
| `lead_time_days` | smallint | yes | Drives task due dates |
| `is_blocking` | boolean | no | Whether an unsatisfied requirement blocks departure |
| `effective_from` / `effective_to` | date | no / yes | Rules change; history must be readable |

- **Unique** (`destination_id`, `nationality_code`, `document_kind`, `effective_from`)
- **Indexes** (`destination_id`, `nationality_code`) — read whenever requirements are computed
- **Check** `effective_to IS NULL OR effective_to > effective_from`
- **Soft delete** No — superseded by a new `effective_from` row
- **Note** This table stores the *policy*. It does **not** store which requirement is satisfied — that is derived (§5).

---

## 3.3 `crm`

---

**T7 · `crm.families`** — the account. Aggregate root.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `display_name` | text | no | "The Mehtas" |
| `state` | family_state | no | `prospective`, `active`, `dormant`, `archived` |
| `relationship_owner_id` | uuid | no | → operators. Drives consultant RLS. |
| `tier` | family_tier | yes | |
| `principal_traveller_id` | uuid | yes | → travellers. Deferrable FK — a family and its first member are created together. |
| `merged_into_family_id` | uuid | yes | Set when this family was merged away; makes redirects possible |
| `search_vector` | tsvector | no | Generated from display name |
| `notes` | text | yes | |

- **PK** `id` · **FK** `relationship_owner_id → operators` RESTRICT; `principal_traveller_id → travellers` DEFERRABLE INITIALLY DEFERRED; `merged_into_family_id → families` RESTRICT
- **Unique** none on name — two families may legitimately be "The Shahs"
- **Indexes** `relationship_owner_id` (RLS hot path, partial `WHERE deleted_at IS NULL`); `state`; GIN on `search_vector`; GIN trigram on `display_name` for fuzzy matching during intake
- **Check** `state = 'archived'` requires `deleted_at IS NULL` — archived is a business state, not a deletion
- **Soft delete** Yes
- **Lifecycle** prospective → active → dormant → archived
- **Invariant 2** (exactly one principal contact) is enforced by `principal_traveller_id` being set for any family past `prospective`, plus a composite FK ensuring the traveller belongs to this family.

---

**T8 · `crm.travellers`** — a person who travels. Entity **inside** the Family aggregate.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `family_id` | uuid | no | → families. Invariant 1: exactly one. |
| `given_name` / `family_name` | text | no / no | |
| `date_of_birth` | date | yes | Drives minor status; nullable because it arrives late |
| `nationality_code` | char(2) | yes | → countries. Drives document policy. |
| `family_role` | family_role | no | `principal`, `payer`, `member`, `minor`, `elder`, `assistant` |
| `responsible_adult_id` | uuid | yes | → travellers. Required when minor (invariant 4). |
| `state` | traveller_state | no | `draft`, `complete`, `inactive` |
| `search_vector` | tsvector | no | |

- **PK** `id` · **FK** `family_id → families` **CASCADE** (inside the aggregate); `responsible_adult_id → travellers` RESTRICT
- **Unique** **(`family_id`, `id`)** — a redundant-looking unique key that exists so other tables can use a composite FK to prove a traveller belongs to a given family. This is the mechanism that makes invariant 6 database-enforceable rather than application-hoped.
- **Indexes** `family_id`; GIN trigram on `given_name`, `family_name` (duplicate detection is a named domain risk); `state`
- **Check** `family_role = 'minor'` implies `responsible_adult_id IS NOT NULL`
- **Soft delete** Yes
- **Lifecycle** draft → complete → inactive

---

**T9 · `crm.contact_identities`** — how to reach, and how to *recognise*, a family or traveller on a channel.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `family_id` | uuid | no | → families |
| `traveller_id` | uuid | yes | Null = belongs to the family generally |
| `channel` | channel | no | `email`, `phone`, `whatsapp` |
| `identifier` | citext | no | Normalised: lowercase email, E.164 phone |
| `is_primary` | boolean | no | |
| `verified_at` | timestamptz | yes | |

- **PK** `id` · **FK** composite (`family_id`, `traveller_id`) → `travellers(family_id, id)` RESTRICT
- **Unique** (`channel`, `identifier`) partial `WHERE deleted_at IS NULL` — one identity maps to one family, which is exactly what makes intake matching deterministic
- **Indexes** the unique index above serves sender lookup on every inbound message
- **Soft delete** Yes
- **Why it matters now and later** This is how an inbound email is matched to a family today, and how a WhatsApp number is matched to a family in a year. The `channel` column is the entire WhatsApp migration for this table.

---

**T10 · `crm.preferences`** — structured preferences at family or traveller level.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `family_id` | uuid | no | → families |
| `traveller_id` | uuid | yes | Null = family-level preference |
| `category` | preference_category | no | `accommodation`, `flight`, `dietary`, `accessibility`, `experience`, `avoid`, `celebration` |
| `key` | text | no | e.g. `cabin_class` |
| `value` | text | no | e.g. `business` |
| `strength` | preference_strength | no | `always`, `prefers`, `avoids`, `never` |
| `learned_from_trip_id` | uuid | yes | → trips. Where the agency learned it. |

- **PK** `id` · **FK** composite (`family_id`, `traveller_id`) → `travellers(family_id, id)` — the one deliberate composite cross-reference, because a preference attached to the wrong family is a data-integrity failure with client-visible consequences
- **Unique** (`family_id`, `traveller_id`, `category`, `key`) partial `WHERE deleted_at IS NULL`
- **Indexes** (`family_id`, `category`)
- **Soft delete** Yes — a withdrawn preference is history worth keeping
- **Note** Trip-level preferences (the third level in the Domain Model) live on the trip, not here; they are not durable family memory.

---

**T11 · `crm.family_links`** — peer relationships between families for group and multi-generational travel.

`id` uuid PK · `family_a_id` uuid → families · `family_b_id` uuid → families · `relation` family_relation (`extended_family`, `travels_with`, `family_office`) · `note` text

- **Unique** (`family_a_id`, `family_b_id`) · **Check** `family_a_id < family_b_id` — canonical ordering so a link cannot be stored twice
- **Indexes** both columns · **Soft delete** Yes
- **Note** Present from day one specifically because the Family boundary is the model's most likely fracture point.

---

**T12 · `crm.family_merges`** — the record of a merge. Append-only.

`id` uuid PK · `surviving_family_id` uuid → families · `merged_family_id` uuid → families · `performed_by` uuid → operators · `performed_at` timestamptz · `summary` jsonb (what moved)

- **Unique** `merged_family_id` — a family can only be merged away once
- **Soft delete** No, append-only. The merged family row survives soft-deleted with `merged_into_family_id` set, so old references resolve.

---

## 3.4 `pipeline`

---

**T13 · `pipeline.enquiries`** — a stated intention to travel. Aggregate root.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `family_id` | uuid | no | → families (may be a newly created prospective family) |
| `owner_id` | uuid | no | → operators |
| `state` | enquiry_state | no | `captured`, `qualified`, `in_design`, `won`, `lost`, `archived` |
| `source` | enquiry_source | no | `email`, `whatsapp`, `phone`, `referral`, `instagram`, `walk_in`, `repeat` |
| `source_message_id` | uuid | yes | → `intake.inbound_messages`. Provenance for mail-borne enquiries. |
| `requested_start` / `requested_end` | date | yes | Often unknown at capture |
| `party_adults` / `party_children` | smallint | yes | |
| `budget_band` | budget_band | yes | |
| `budget_amount` / `budget_currency` | numeric(14,2) / char(3) | yes | |
| `brief` | text | yes | The client's own words |
| `loss_reason_id` | uuid | yes | → loss_reasons. Required when lost. |
| `qualified_at`, `won_at`, `lost_at` | timestamptz | yes | |
| `source_review_item_id` | uuid | yes | Provenance |

- **PK** `id` · **FK** family RESTRICT, owner RESTRICT, loss reason RESTRICT
- **Indexes** (`owner_id`, `state`) partial on open states — the consultant's main list; `family_id`; `created_at` for ageing
- **Check** `state = 'lost'` requires `loss_reason_id IS NOT NULL` (loss reasons are how the agency learns); budget amount and currency both null or both set
- **Soft delete** Yes
- **Lifecycle** captured → qualified → in_design → won | lost → archived. **No `trip_id` column** — the Trip references the Enquiry, not the reverse, because the Enquiry survives as the record of demand and must not be mutated by a downstream context.

---

**T14 · `pipeline.loss_reasons`** — lookup with attributes (category, whether it counts as competitive loss), hence a table rather than an enum.

`id` uuid PK · `code` text UNIQUE · `label` text · `category` text · `is_competitive` boolean · `sort_order` smallint · `active` boolean

- **Soft delete** No — deactivate via `active`

---

## 3.5 `planning`

---

**T15 · `planning.trips`** — one journey for one family. Aggregate root, and the centre of the model.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `family_id` | uuid | no | → families |
| `enquiry_id` | uuid | yes | → enquiries. Null for trips that began without one. |
| `reference` | text | no | Human-facing trip code, e.g. `PL-2026-0184` |
| `title` | text | no | "Mehta Family — Maldives, Aug 2026" |
| `state` | trip_state | no | `draft`, `proposed`, `confirmed`, `in_progress`, `completed`, `cancelled`, `archived` |
| `owner_id` | uuid | no | → operators (consultant) |
| `ops_owner_id` | uuid | yes | → operators. Set at confirmation. |
| `start_date` / `end_date` | date | no / no | |
| `travel_span` | daterange | no | Generated from start/end. Used by exclusion constraints and overlap queries. |
| `accepted_proposal_id` | uuid | yes | → proposals. Deferrable. At most one (invariant 12). |
| `confirmed_at`, `cancelled_at`, `completed_at` | timestamptz | yes | |
| `cancellation_reason` | text | yes | |

- **PK** `id` · **FK** family RESTRICT, enquiry RESTRICT, owners RESTRICT, accepted proposal DEFERRABLE
- **Unique** `reference` · **Unique** (`id`, `family_id`) — again a composite key, so bookings and party rows can prove they belong to the same family
- **Indexes** (`owner_id`, `state`); (`ops_owner_id`, `state`) partial on live states; `family_id`; GiST on `travel_span` for "what departs in August"; `state` partial `WHERE state IN ('confirmed','in_progress')`
- **Check** `end_date >= start_date`; `state = 'confirmed'` requires `accepted_proposal_id IS NOT NULL` (invariant 10) and `confirmed_at IS NOT NULL`
- **Soft delete** Yes
- **Lifecycle** draft → proposed → confirmed → in_progress → completed → archived, with cancelled reachable from proposed/confirmed/in_progress
- **Note** No `margin`, `total_cost` or `readiness` columns. All derived (§5).

---

**T16 · `planning.trip_party`** — which travellers are on the trip.

`trip_id` uuid → trips · `family_id` uuid · `traveller_id` uuid · `is_lead` boolean

- **PK** (`trip_id`, `traveller_id`)
- **FK** `trip_id → trips` CASCADE (inside the aggregate); **composite (`family_id`, `traveller_id`) → `travellers(family_id, id)`** and **composite (`trip_id`, `family_id`) → `trips(id, family_id)`** — together these make **invariant 6 structurally impossible to violate**: a traveller from another family cannot be added to this trip, at any layer, by any code path.
- **Indexes** `traveller_id` (reverse lookup: "which trips is this person on?")
- **Soft delete** No — hard delete, cascading from trip

---

**T17 · `planning.trip_destinations`** — where the trip goes. Drives document policy and reporting.

`trip_id` uuid → trips CASCADE · `destination_id` uuid → destinations · `arrival_date` date · `departure_date` date · `sequence` smallint

- **PK** (`trip_id`, `destination_id`, `sequence`)
- **Indexes** `destination_id` (demand analysis)
- **Check** `departure_date >= arrival_date`; both within the trip's `travel_span`
- **Soft delete** No

---

**T18 · `planning.itinerary_days`** — the day structure of a trip.

`id` uuid PK · `trip_id` uuid → trips CASCADE · `day_date` date · `day_number` smallint · `title` text · `narrative` text

- **Unique** (`trip_id`, `day_date`); (`trip_id`, `day_number`)
- **Check** `day_date` contained in the trip's `travel_span` (invariant 7)
- **Indexes** `trip_id`
- **Soft delete** No — hard delete inside the aggregate

---

**T19 · `planning.itinerary_components`** — one sellable element. Entity inside the Trip aggregate.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `trip_id` | uuid | no | → trips CASCADE |
| `day_id` | uuid | yes | → itinerary_days. Null for multi-day stays anchored by date range. |
| `kind` | component_kind | no | `accommodation`, `flight`, `transfer`, `experience`, `free_day`, `other` |
| `selection_state` | component_selection_state | no | `proposed`, `selected`, `removed` — **planning-owned states only** |
| `property_id` | uuid | yes | → properties |
| `supplier_id` | uuid | yes | → suppliers |
| `room_type_id` | uuid | yes | → property_room_types |
| `stay_span` | daterange | yes | For accommodation |
| `starts_at` / `ends_at` | timestamptz | yes | For flights and transfers |
| `description` | text | yes | |
| `rate_line_id` | uuid | yes | → rate_lines. Reference: where the price came from. |
| `unit_sell` / `sell_currency` | numeric(14,2) / char(3) | yes | Copied at selection |
| `quantity` | smallint | no | Default 1 |
| `rate_captured_at` | timestamptz | yes | When the price was copied (invariant 20) |
| `sort_order` | smallint | no | |

- **PK** `id` · **FK** trip CASCADE; property, supplier, room type, rate line all RESTRICT
- **Unique** none · **Indexes** (`trip_id`, `sort_order`); `property_id`; `rate_line_id` (contract usage analysis); GiST on (`trip_id`, `stay_span`) for the overlap constraint
- **Exclusion constraint** For `kind = 'accommodation'` and `selection_state = 'selected'`, no two components on the same trip may have overlapping `stay_span` for the same traveller set — **invariant 8 enforced by the database**, using a GiST exclusion constraint rather than trusted application code.
- **Check** accommodation requires `stay_span` and `property_id`; flight requires `starts_at`; `rate_line_id IS NOT NULL` requires `rate_captured_at IS NOT NULL`
- **Soft delete** No — `selection_state = 'removed'` is the business form of removal, and hard delete cascades only with the trip
- **Note** **There is no `booked` or `delivered` state here.** The Domain Model says component state *reflects* its booking; storing it would create a second source of truth. Booked status is derived by joining `booking.bookings`.

---

**T20 · `planning.component_costs`** — supplier cost per component. **A satellite table, split for access control.**

`component_id` uuid PK → components CASCADE · `unit_cost` numeric(14,2) · `cost_currency` char(3) · `cost_source` cost_source (`contract`, `manual`, `estimate`)

- **PK** `component_id` (one-to-one)
- **Why split.** Supabase connects every signed-in operator as the same PostgreSQL role, so column-level `GRANT`s cannot distinguish a consultant from an operations executive. **Row Level Security is the only per-user mechanism available, and RLS is row-level.** Putting cost in its own table is therefore the only clean way to let Rahul see a trip without seeing its margin (§7). This is a real Supabase constraint driving a real schema decision, not premature normalisation.
- **Indexes** none needed beyond PK
- **Soft delete** No — cascades with the component

---

**T21 · `planning.proposals`** — an immutable snapshot of what the client was shown. Aggregate root.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `trip_id` | uuid | no | → trips RESTRICT (cross-aggregate: no cascade) |
| `version_no` | smallint | no | |
| `state` | proposal_state | no | `draft`, `sent`, `accepted`, `superseded`, `declined` |
| `snapshot` | jsonb | no | Whole itinerary and pricing, frozen |
| `snapshot_hash` | text | no | Tamper-evidence and duplicate detection |
| `total_sell` / `currency` | numeric(14,2) / char(3) | no | Denormalised **deliberately** — this is a historical fact, not a cached join |
| `sent_at`, `sent_to`, `accepted_at`, `declined_at` | timestamptz / text | yes | |
| `rendered_file_path` | text | yes | Supabase Storage path of the PDF actually sent |

- **PK** `id` · **Unique** (`trip_id`, `version_no`); **partial unique** `trip_id WHERE state = 'accepted'` — invariant 12 enforced by the database
- **Indexes** (`trip_id`, `version_no` DESC)
- **Check** `state <> 'draft'` requires `sent_at IS NOT NULL`
- **Immutability** Trigger rejects `UPDATE` to `snapshot`, `total_sell` or `version_no` once `state <> 'draft'` (invariant 11). State may still advance; content may not.
- **Soft delete** No — append-only after sending

---

## 3.6 `rates`

---

**T22 · `rates.suppliers`** — anyone the agency buys from.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `name` | text | no | |
| `kind` | supplier_kind | no | `hotel`, `dmc`, `transport`, `guide`, `experience`, `airline`, `other` |
| `state` | supplier_state | no | `prospective`, `active`, `preferred`, `suspended`, `archived` |
| `country_code` | char(2) | yes | → countries |
| `payment_terms` | text | yes | |
| `default_currency` | char(3) | yes | |
| `search_vector` | tsvector | no | |

- **Unique** `lower(name)` partial `WHERE deleted_at IS NULL`
- **Indexes** `state`; GIN `search_vector`; GIN trigram on `name`
- **Soft delete** Yes · **Lifecycle** prospective → active → preferred → suspended → archived
- **Note** No `performance_score` column — derived from bookings and tasks (§5).

---

**T23 · `rates.supplier_contacts`** — named people and channel identifiers at a supplier. Mirrors `crm.contact_identities` in purpose: this is how an inbound supplier email is matched.

`id` uuid PK · `supplier_id` uuid → suppliers CASCADE · `name` text · `role` text · `channel` channel · `identifier` citext · `is_primary` boolean

- **Unique** (`channel`, `identifier`) partial `WHERE deleted_at IS NULL` · **Indexes** the same index serves intake sender lookup
- **Soft delete** Yes

---

**T24 · `rates.properties`** — a specific sellable place.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `name` | text | no | |
| `destination_id` | uuid | no | → destinations |
| `category` | property_category | yes | `hotel`, `resort`, `villa`, `camp`, `lodge`, `palace` |
| `star_rating` | smallint | yes | |
| `state` | property_state | no | `draft`, `published`, `under_review`, `retired` |
| `description` | text | yes | |
| `search_vector` | tsvector | no | Generated from name, destination, category |

- **Unique** (`destination_id`, `lower(name)`) partial `WHERE deleted_at IS NULL`
- **Indexes** `destination_id`; GIN `search_vector`; GIN trigram on `name` — **the rate-search hot path**
- **Check** `star_rating BETWEEN 1 AND 7`
- **Soft delete** Yes

---

**T25 · `rates.property_room_types`** — the room categories a property sells.

`id` uuid PK · `property_id` uuid → properties CASCADE · `name` text · `max_occupancy` smallint · `sort_order` smallint

- **Unique** (`property_id`, `lower(name)`) partial `WHERE deleted_at IS NULL` · **Indexes** `property_id`
- **Soft delete** Yes

---

**T26 · `rates.supplier_properties`** — which suppliers can sell which properties. Many-to-many, because the same villa may be bought direct or through two DMCs at different rates — a commercially significant fact the Domain Model calls out explicitly.

`supplier_id` uuid · `property_id` uuid · `is_direct` boolean · `notes` text

- **PK** (`supplier_id`, `property_id`) · **Indexes** `property_id`
- **Soft delete** No — hard delete; the relationship either exists or does not

---

**T27 · `rates.rate_contracts`** — a negotiated agreement. Aggregate root. **The strategic asset.**

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `supplier_id` | uuid | no | → suppliers RESTRICT |
| `reference` | text | yes | Supplier's contract reference |
| `state` | rate_contract_state | no | `draft`, `under_review`, `active`, `expiring`, `expired`, `superseded` |
| `validity` | daterange | no | Contract validity window |
| `currency` | char(3) | no | |
| `commission_basis` | commission_basis | no | `net`, `commissionable` |
| `commission_pct` | numeric(5,2) | yes | |
| `cancellation_terms` | text | yes | |
| `blackout_dates` | daterange[] | yes | |
| `superseded_by_id` | uuid | yes | → rate_contracts (invariant 19) |
| `source_file_path` | text | yes | Supabase Storage path of the original PDF |
| `source_message_id` | uuid | yes | → inbound_messages — where it arrived from |
| `source_review_item_id` | uuid | yes | Provenance: who accepted it |
| `steward_id` | uuid | yes | → operators. Who is responsible for freshness. |

- **PK** `id` · **FK** supplier RESTRICT; `superseded_by_id` RESTRICT
- **Unique** (`supplier_id`, `reference`) partial `WHERE reference IS NOT NULL AND deleted_at IS NULL`
- **Indexes** (`supplier_id`, `state`); GiST on `validity`; `state` partial `WHERE state IN ('active','expiring')` — **quotability is checked on every rate search**
- **Check** `state = 'superseded'` requires `superseded_by_id IS NOT NULL`; `commission_basis = 'commissionable'` requires `commission_pct IS NOT NULL`
- **Soft delete** Yes, but expiry and supersession are the normal ends
- **Lifecycle** draft → under_review → active → expiring → expired | superseded
- **Note** **No editing after activation.** Amendments create a new contract with `superseded_by_id` pointing back. Enforced by trigger on commercial columns.

---

**T28 · `rates.rate_lines`** — one priced row. Entity inside the Rate Contract aggregate.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `contract_id` | uuid | no | → rate_contracts CASCADE |
| `property_id` | uuid | no | → properties RESTRICT |
| `room_type_id` | uuid | yes | → property_room_types RESTRICT |
| `season` | daterange | no | |
| `occupancy_basis` | occupancy_basis | no | `single`, `double`, `triple`, `child`, `per_unit` |
| `net_amount` / `published_amount` | numeric(14,2) | yes | At least one required |
| `currency` | char(3) | no | |
| `min_nights` | smallint | yes | |
| `inclusions` | text[] | yes | Structured, not prose (Vision M6) |
| `meal_plan` | meal_plan | yes | `room_only`, `bb`, `hb`, `fb`, `ai` |

- **PK** `id` · **Unique** (`contract_id`, `property_id`, `room_type_id`, `season`, `occupancy_basis`) — one price per cell
- **Indexes** **(`property_id`, `season`) GiST** — the primary rate-search index; (`contract_id`); (`room_type_id`)
- **Check** `net_amount IS NOT NULL OR published_amount IS NOT NULL`; `season` contained within the contract's `validity` (enforced by trigger, since a check constraint cannot reach the parent row)
- **Soft delete** No — cascades with its contract, which is never edited after activation

---

## 3.7 `booking`

---

**T29 · `booking.bookings`** — a commitment made with a supplier. Aggregate root, deliberately outside `planning`.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `trip_id` | uuid | no | → trips RESTRICT (invariant 13) |
| `component_id` | uuid | yes | → components RESTRICT. At most one booking per component. |
| `supplier_id` | uuid | no | → suppliers RESTRICT |
| `state` | booking_state | no | `requested`, `held`, `confirmed`, `delivered`, `cancelled`, `lapsed` |
| `supplier_reference` | text | yes | Required to confirm (invariant 14) |
| `hold_expires_at` | timestamptz | yes | Required when held (invariant 15) |
| `cancellation_deadline` | timestamptz | yes | |
| `cancellation_terms` | text | yes | Required to cancel a confirmed booking (invariant 16) |
| `agreed_cost` / `cost_currency` | numeric(14,2) / char(3) | yes | Ops-visible: needed to pay suppliers |
| `confirmed_at`, `cancelled_at`, `lapsed_at` | timestamptz | yes | |
| `source_review_item_id` | uuid | yes | Provenance — many confirmations arrive by email |

- **PK** `id` · **Unique** `component_id` partial `WHERE component_id IS NOT NULL AND state <> 'cancelled'` — a component is realised by at most one live booking
- **Indexes** (`trip_id`, `state`); (`supplier_id`, `state`); **`hold_expires_at` partial `WHERE state = 'held'`** — the index that stops holds from lapsing unnoticed; `cancellation_deadline` partial on live bookings
- **Check** `state = 'confirmed'` requires `supplier_reference IS NOT NULL`; `state = 'held'` requires `hold_expires_at IS NOT NULL`; `state = 'lapsed'` requires `lapsed_at IS NOT NULL`
- **Soft delete** No. `cancelled` and `lapsed` are business states and must remain visible — **`lapsed` in particular is an operational failure that reporting needs to count.**
- **Lifecycle** requested → held → confirmed → delivered, with cancelled from most states and lapsed only from held

---

**T30 · `booking.booking_travellers`** — which travellers a booking names.

`booking_id` uuid → bookings CASCADE · `traveller_id` uuid → travellers RESTRICT

- **PK** (`booking_id`, `traveller_id`) · **Indexes** `traveller_id`
- **Soft delete** No

---

## 3.8 `sabre`

Everything Sabre-shaped stops at this schema boundary. No table outside `sabre` stores a record locator, a segment status code, or a Sabre passenger name.

---

**T31 · `sabre.reservations`** — the local mirror of a PNR. **Never a master.**

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `record_locator` | text | no | Sabre's key — an attribute, never our identity |
| `sync_state` | sync_state | no | `unmatched`, `in_sync`, `diverged`, `resolved`, `closed` |
| `booking_id` | uuid | yes | → bookings RESTRICT. Null = unmatched, a legitimate resting state. |
| `trip_id` | uuid | yes | → trips RESTRICT. Denormalised for queue filtering; set only with `booking_id`. |
| `raw_payload` | jsonb | no | What Sabre returned, unmodified |
| `payload_hash` | text | no | Cheap divergence detection |
| `ticketing_deadline` | timestamptz | yes | |
| `last_checked_at` | timestamptz | no | Drives staleness |
| `diverged_at`, `resolved_at`, `resolved_by` | timestamptz / uuid | yes | |
| `divergence_summary` | jsonb | yes | What disagrees |

- **PK** `id` · **Unique** `record_locator` partial `WHERE sync_state <> 'closed'`
- **Indexes** `sync_state` partial `WHERE sync_state IN ('unmatched','diverged')` — the ops work queue; `last_checked_at` partial on open records (staleness sweep); `booking_id`; `ticketing_deadline` partial where not null
- **Check** `sync_state = 'in_sync'` requires `booking_id IS NOT NULL`; `booking_id IS NULL` requires `trip_id IS NULL`
- **Soft delete** No — `closed` is the terminal state
- **Lifecycle** discovered → matched → in_sync ⇄ diverged → resolved → closed
- **Note** `raw_payload` is kept because reconciliation arguments are settled by evidence, not by our interpretation of it.

---

**T32 · `sabre.reservation_segments`** — flight segments as Sabre reports them.

`id` uuid PK · `reservation_id` uuid → reservations CASCADE · `sequence` smallint · `carrier` text · `flight_number` text · `depart_airport` char(3) · `arrive_airport` char(3) · `departs_at` timestamptz · `arrives_at` timestamptz · `status_code` text · `cabin` text

- **Unique** (`reservation_id`, `sequence`) · **Indexes** `reservation_id`; `departs_at`
- **Soft delete** No — replaced wholesale on each sync

---

**T33 · `sabre.reservation_passengers`** — Sabre passengers, and our attempt to map them to travellers.

`id` uuid PK · `reservation_id` uuid → reservations CASCADE · `sabre_name` text · `traveller_id` uuid → travellers RESTRICT (nullable) · `match_confidence` numeric(3,2) · `matched_by` uuid → operators · `matched_at` timestamptz

- **Unique** (`reservation_id`, `sabre_name`)
- **Indexes** `traveller_id`; partial `WHERE traveller_id IS NULL` — unmapped passengers are review-queue work
- **Note** A null `traveller_id` is expected and visible, never silently guessed. Mapping is a human decision (rule 34).
- **Soft delete** No

---

**T34 · `sabre.sync_runs`** — append-only log of sync attempts. Answers "is this current?" and "when did it last work?"

`id` uuid PK · `started_at` / `finished_at` timestamptz · `trigger` sync_trigger (`scheduled`, `manual`, `event`) · `reservations_checked` int · `divergences_found` int · `errors` jsonb · `outcome` sync_outcome

- **Indexes** `started_at DESC` · **Soft delete** No, append-only · **Partition candidate** by month, once volume warrants

---

## 3.9 `documents`

The most sensitive schema in the system. Strictest RLS, mandatory access logging, hard retention.

---

**T35 · `documents.documents`** — evidence about a person. Aggregate root.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `traveller_id` | uuid | no | → travellers RESTRICT (invariant 21) |
| `kind` | document_kind | no | `passport`, `visa`, `insurance`, `id`, `other` |
| `state` | document_state | no | `requested`, `received`, `verified`, `valid`, `expiring`, `expired`, `rejected`, `purged` |
| `file_path` | text | yes | Supabase Storage path. Null when requested or purged. |
| `file_hash` | text | yes | Duplicate detection |
| `detail` | jsonb | yes | Typed per kind: passport number, nationality, place of issue |
| `document_number` | text | yes | Promoted out of `detail` for uniqueness and lookup |
| `issued_on` / `expires_on` | date | yes | |
| `issuing_country` | char(2) | yes | → countries |
| `verified_by` / `verified_at` | uuid / timestamptz | yes | A human checked the extraction against the image |
| `requested_at` | timestamptz | yes | |
| `retention_until` | date | yes | Drives purge |
| `purged_at` / `purged_by` | timestamptz / uuid | yes | |
| `source_message_id` | uuid | yes | → inbound_messages |
| `source_review_item_id` | uuid | yes | Provenance |

- **PK** `id` · **FK** traveller RESTRICT — deleting a traveller must never silently destroy their passport record
- **Unique** (`traveller_id`, `kind`, `document_number`) partial `WHERE document_number IS NOT NULL AND deleted_at IS NULL`
- **Indexes** `traveller_id`; **`expires_on` partial `WHERE state IN ('valid','expiring')`** — the index behind "whose passport expires before this trip"; `state` partial `WHERE state = 'requested'` (the chasing list); `retention_until` partial where not null
- **Check** `state = 'valid'` requires `expires_on IS NOT NULL` for kinds that expire (invariant 22); `state = 'purged'` requires `file_path IS NULL AND purged_at IS NOT NULL`; `state IN ('received','verified','valid')` requires `file_path IS NOT NULL`
- **Soft delete** Yes for withdrawal, but **purge is different**: the stored object is hard-deleted, `file_path` and `detail` are nulled, and the row survives as a tombstone (invariant 24). A soft-deleted document is hidden; a purged one is gone.
- **Lifecycle** requested → received → verified → valid → expiring → expired → purged, with rejected reachable from received
- **Note** `requested` documents have no file. This is deliberate — an unfulfilled request is a business fact that drives chasing.

---

**T36 · `documents.document_access_log`** — every read of a document. Append-only, no exceptions (invariant 23).

`id` uuid PK · `document_id` uuid → documents RESTRICT · `operator_id` uuid → operators · `accessed_at` timestamptz · `action` document_action (`viewed`, `downloaded`, `metadata_only`) · `context` text · `ip_hash` text

- **Indexes** (`document_id`, `accessed_at DESC`); (`operator_id`, `accessed_at DESC`)
- **Soft delete** No. Insert-only for every role including administrators. **Partition candidate** by month.

---

## 3.10 `ops`

---

**T37 · `ops.task_templates`** — the rules that generate tasks. A table rather than code, because Sameer must be able to see and adjust them without a deploy.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `code` | text | no | Unique |
| `label` | text | no | |
| `trigger_event` | text | no | e.g. `TripConfirmed`, `BookingHeld` |
| `subject_kind` | task_subject_kind | no | What the task attaches to |
| `offset_days` | smallint | yes | Relative to an anchor date |
| `anchor` | task_anchor | yes | `trip_start`, `trip_end`, `hold_expiry`, `deadline`, `now` |
| `default_role` | operator_role | yes | Who it goes to |
| `is_blocking` | boolean | no | Blocks departure |
| `conditions` | jsonb | yes | e.g. only for international destinations |
| `active` | boolean | no | |

- **Unique** `code` · **Indexes** (`trigger_event`, `active`)
- **Soft delete** No — deactivate

---

**T38 · `ops.tasks`** — a unit of work with a deadline. Aggregate root.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `template_id` | uuid | yes | → task_templates. Null for manually created tasks (the exception). |
| `title` | text | no | |
| `state` | task_state | no | `open`, `in_progress`, `completed`, `cancelled` |
| `subject_kind` | task_subject_kind | no | `trip`, `family`, `booking`, `document`, `rate_contract`, `reservation` |
| `trip_id` | uuid | yes | → trips RESTRICT |
| `family_id` | uuid | yes | → families RESTRICT |
| `booking_id` | uuid | yes | → bookings RESTRICT |
| `document_id` | uuid | yes | → documents RESTRICT |
| `rate_contract_id` | uuid | yes | → rate_contracts RESTRICT |
| `reservation_id` | uuid | yes | → reservations RESTRICT |
| `assignee_id` | uuid | yes | → operators |
| `due_at` | timestamptz | no | |
| `is_blocking` | boolean | no | |
| `completed_at` / `completed_by` | timestamptz / uuid | yes | |
| `cancelled_reason` | text | yes | |

- **PK** `id` · **Check** exactly one subject FK is non-null and matches `subject_kind` — **typed nullable FKs, not a polymorphic `subject_id`**, so referential integrity is real. Six nullable columns is a small price for the database enforcing that every task points at something that exists.
- **Indexes** **(`assignee_id`, `state`, `due_at`) partial `WHERE state IN ('open','in_progress')`** — Rahul's daily screen, the single most-run query in the product; `trip_id` partial on open; `due_at` partial on open (the overdue sweep); (`state`, `is_blocking`) partial for readiness
- **Soft delete** Yes
- **Lifecycle** generated → open → in_progress → completed | cancelled
- **Note** **No `is_overdue` column.** Overdue is `due_at < now() AND state IN ('open','in_progress')` — a condition, not a state (Domain Model, §6).

---

## 3.11 `money`

---

**T39 · `money.payment_schedules`** — what is owed and when. Aggregate root, one per trip.

`id` uuid PK · `trip_id` uuid → trips RESTRICT · `state` schedule_state (`draft`, `active`, `settled`, `written_off`) · `currency` char(3) · `terms_note` text

- **Unique** `trip_id` partial `WHERE deleted_at IS NULL` — one live schedule per trip
- **Indexes** `trip_id`; `state`
- **Soft delete** Yes
- **Note** No `outstanding_balance` column — derived (§5, invariant 27).

---

**T40 · `money.payment_milestones`** — one expected movement of money.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `schedule_id` | uuid | no | → payment_schedules CASCADE |
| `direction` | payment_direction | no | `from_client`, `to_supplier` |
| `supplier_id` | uuid | yes | → suppliers. Required when paying a supplier. |
| `label` | text | no | "Deposit", "Final balance" |
| `amount` / `currency` | numeric(14,2) / char(3) | no | |
| `due_on` | date | no | |
| `state` | milestone_state | no | `pending`, `due`, `part_paid`, `paid`, `overdue`, `waived` |
| `sequence` | smallint | no | |

- **Unique** (`schedule_id`, `sequence`) · **Indexes** **`due_on` partial `WHERE state IN ('pending','due','part_paid')`** — the chasing index; (`schedule_id`); (`supplier_id`, `due_on`) partial for supplier payables
- **Check** `amount > 0`; `direction = 'to_supplier'` requires `supplier_id IS NOT NULL`
- **Soft delete** No — `waived` is the business form of cancellation
- **Lifecycle** pending → due → part_paid → paid, with overdue and waived reachable

---

**T41 · `money.payments`** — a recorded movement of money. **Append-only.**

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `milestone_id` | uuid | no | → payment_milestones RESTRICT |
| `amount` / `currency` | numeric(14,2) / char(3) | no | Negative amount = a reversal |
| `received_on` | date | no | |
| `method` | payment_method | no | `bank_transfer`, `card`, `cash`, `cheque`, `offset` |
| `reference` | text | yes | |
| `state` | payment_state | no | `recorded`, `confirmed`, `reversed` |
| `reverses_payment_id` | uuid | yes | → payments. Corrections are new rows. |
| `source_message_id` | uuid | yes | Payment advices often arrive by email |
| `source_review_item_id` | uuid | yes | Provenance |

- **PK** `id` · **Indexes** `milestone_id`; `received_on`
- **Check** `amount <> 0`; a row with `reverses_payment_id` set must have the opposite sign
- **Immutability** Trigger rejects `UPDATE` and `DELETE` once `state = 'confirmed'` (invariant 26)
- **Soft delete** No, ever. Financial records are permanent.
- **Note** The platform records that money moved; it never moves it (Domain Model).

---

## 3.12 `intake`

Owns no business record. Every row here is evidence or a proposal.

---

**T42 · `intake.inbound_messages`** — something that arrived. **Immutable.**

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `channel` | channel | no | `email` today; `whatsapp` later. **The whole channel migration is this column.** |
| `external_id` | text | no | Provider message id — dedupe key |
| `mailbox` | text | yes | Which agency mailbox received it |
| `direction` | message_direction | no | `inbound` (outbound lives in `conversations`) |
| `sender_identifier` | citext | no | Normalised, matched against contact identities |
| `sender_name` | text | yes | |
| `recipients` | text[] | yes | |
| `subject` | text | yes | |
| `body_text` / `body_html` | text | yes | |
| `received_at` | timestamptz | no | |
| `state` | message_state | no | `received`, `classified`, `extracted`, `proposed`, `resolved`, `ignored`, `unprocessable` |
| `matched_family_id` | uuid | yes | → families. Best match, human-confirmable. |
| `matched_supplier_id` | uuid | yes | → suppliers |
| `matched_trip_id` | uuid | yes | → trips |
| `search_vector` | tsvector | no | Generated from subject + body |

- **PK** `id` · **Unique** (`channel`, `external_id`) — makes re-ingestion idempotent, which matters because mail polling will duplicate
- **Indexes** `received_at DESC`; `state` partial on unresolved; `sender_identifier`; GIN `search_vector`; `matched_family_id`
- **Immutability** Trigger rejects `UPDATE` to content columns. `state` and `matched_*` may advance.
- **Soft delete** No — evidence is permanent. **Partition candidate** by month (highest-growth table in the system).
- **Privacy note** This table holds client correspondence. It is a first-class RLS and retention concern (§7), not incidental data.

---

**T43 · `intake.message_attachments`** — files that arrived with a message.

`id` uuid PK · `message_id` uuid → inbound_messages CASCADE · `filename` text · `mime_type` text · `size_bytes` bigint · `file_path` text (Storage) · `file_hash` text · `parsed_state` attachment_parse_state (`pending`, `parsed`, `unparseable`, `skipped`)

- **Unique** (`message_id`, `file_hash`) · **Indexes** `message_id`; `parsed_state` partial `WHERE parsed_state = 'pending'`
- **Soft delete** No

---

**T44 · `intake.extractions`** — what the machine believes a message means. **Immutable. Holds no authority.**

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `message_id` | uuid | no | → inbound_messages CASCADE |
| `attachment_id` | uuid | yes | → message_attachments |
| `kind` | extraction_kind | no | `enquiry`, `rate_sheet`, `supplier_quote`, `supplier_confirmation`, `pnr`, `passport`, `visa`, `payment_advice`, `other` |
| `payload` | jsonb | no | Extracted fields |
| `field_confidence` | jsonb | no | Confidence per field, not per document |
| `overall_confidence` | numeric(3,2) | no | |
| `model_name` / `model_version` | text | no | Which model produced this (rule 31) |
| `prompt_version` | text | yes | |
| `state` | extraction_state | no | `produced`, `proposed`, `superseded` |
| `produced_at` | timestamptz | no | |

- **PK** `id` · **Indexes** `message_id`; (`kind`, `produced_at DESC`); `overall_confidence` — accuracy reporting reads this constantly
- **Check** `overall_confidence BETWEEN 0 AND 1`
- **Immutability** Trigger rejects `UPDATE` except to `state`. A re-extraction is a new row that supersedes the old one.
- **Soft delete** No · **Partition candidate** by month, with `inbound_messages`

---

**T45 · `intake.review_queue_items`** — **a proposed change awaiting a human decision.** The trust layer, and structurally the most important table in the database.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `extraction_id` | uuid | yes | → extractions. Null for non-extraction proposals (future agent actions). |
| `message_id` | uuid | yes | → inbound_messages |
| `kind` | review_item_kind | no | Mirrors extraction kinds; **extensible without schema change** |
| `state` | review_item_state | no | `pending`, `accepted`, `applied`, `corrected`, `rejected`, `auto_accepted`, `expired` |
| `risk_class` | risk_class | no | `low`, `standard`, `restricted` — restricted may never auto-accept (rule 33) |
| `target_schema` / `target_table` | text | no | Which context this proposes to change |
| `target_id` | uuid | yes | Null for a proposed *creation* |
| `target_state_hash` | text | yes | **The target's state when the proposal was computed.** The staleness guard. |
| `proposed_change` | jsonb | no | Self-describing payload |
| `corrected_change` | jsonb | yes | What the human actually approved, when it differs |
| `decided_by` | uuid | yes | → operators. Null when auto-accepted. |
| `decided_at` | timestamptz | yes | |
| `auto_accept_policy_id` | uuid | yes | → auto_accept_policies. Required when auto-accepted. |
| `applied_at` | timestamptz | yes | |
| `apply_error` | text | yes | Refusal by the owning context is a legitimate outcome |
| `expires_at` | timestamptz | yes | |
| `assigned_to` | uuid | yes | → operators |

- **PK** `id`
- **FK** extraction, message, operators, policy — **no FK to the target row.** Deliberate: a review item must be able to outlive, or be invalidated by, what it targets, and it must be able to propose creating something that does not exist yet.
- **Indexes** **(`state`, `risk_class`, `created_at`) partial `WHERE state = 'pending'`** — the queue itself, the second-most-run query in the product; (`assigned_to`, `state`) partial pending; (`kind`, `state`); (`target_schema`, `target_table`, `target_id`); `state` partial `WHERE state = 'accepted'` — **items accepted but never applied are defects and must be trivially findable**
- **Check** `state = 'auto_accepted'` requires `auto_accept_policy_id IS NOT NULL`; `state IN ('accepted','corrected','rejected')` requires `decided_by IS NOT NULL` — **rule 32 enforced by the database**: no decided item without either a human or a named policy; `risk_class = 'restricted'` forbids `state = 'auto_accepted'` — **rule 33 enforced by the database**
- **Soft delete** No — the decision record is permanent
- **Lifecycle** raised → pending → accepted → applied | corrected → applied | rejected | auto_accepted → applied | expired
- **Notes** `accepted` and `applied` are separate states because human decision and context application can fail independently. `target_state_hash` exists solely to implement policy 46: if the target has moved since the proposal was computed, the item expires rather than applying. This is the model's sharpest concurrency hazard, and it is guarded here.

---

**T46 · `intake.auto_accept_policies`** — the named policies that may accept a proposal without a human (rule 32).

`id` uuid PK · `kind` review_item_kind · `max_risk_class` risk_class · `min_confidence` numeric(3,2) · `conditions` jsonb · `active` boolean · `owner_id` uuid → operators · `activated_at` / `deactivated_at` timestamptz

- **Unique** `kind` partial `WHERE active` — one active policy per kind, so authority is never ambiguous
- **Check** `max_risk_class <> 'restricted'` — **the schema itself refuses to allow a policy that could auto-accept money, identity documents or rates**
- **Soft delete** No — deactivate, never delete; every change is an audited event (Domain Model risk: "auto-accept will quietly widen")

---

## 3.13 `conversations`

---

**T47 · `conversations.threads`** — a conversation about something.

`id` uuid PK · `subject_kind` thread_subject_kind (`family`, `trip`, `supplier`, `booking`) · `family_id` / `trip_id` / `supplier_id` / `booking_id` uuid (typed nullable FKs, one non-null) · `title` text · `last_message_at` timestamptz · `is_internal` boolean

- **Check** exactly one subject FK non-null, matching `subject_kind`
- **Indexes** (`trip_id`, `last_message_at DESC`); (`family_id`, `last_message_at DESC`); `supplier_id`
- **Soft delete** Yes

---

**T48 · `conversations.communications`** — an exchange with someone outside the agency. Append-only.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `thread_id` | uuid | no | → threads RESTRICT |
| `direction` | comm_direction | no | `inbound`, `outbound` |
| `channel` | channel | no | |
| `inbound_message_id` | uuid | yes | → inbound_messages. **The join between the two readings of one arrival.** |
| `participant_identifier` | citext | no | |
| `body` | text | no | |
| `state` | comm_state | no | `drafted`, `awaiting_approval`, `approved`, `sent`, `delivered`, `received`, `filed` |
| `drafted_by_model` | text | yes | Non-null = machine-drafted |
| `approved_by` / `approved_at` | uuid / timestamptz | yes | |
| `sent_by` / `sent_at` | uuid / timestamptz | yes | |
| `occurred_at` | timestamptz | no | |

- **Indexes** (`thread_id`, `occurred_at DESC`); `inbound_message_id`; `state` partial `WHERE state = 'awaiting_approval'`
- **Check** **`drafted_by_model IS NOT NULL AND state = 'sent'` requires `approved_by IS NOT NULL`** — rule 37 enforced by the database *now*, years before the feature exists, because it is the rule that must never become negotiable
- **Soft delete** No, append-only · **Partition candidate** by quarter, eventually

---

**T49 · `conversations.notes`** — internal remarks. Distinct from communications: notes never leave the building.

`id` uuid PK · `subject_kind` / typed nullable FKs (as threads) · `body` text · `author_id` uuid → operators · `is_handover` boolean

- **Indexes** (`trip_id`, `created_at DESC`); (`family_id`, `created_at DESC`)
- **Soft delete** Yes

---

## 3.14 `events` and `audit`

---

**T50 · `events.domain_events`** — the append-only domain event log. Written in the same transaction as the change that caused it.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK (UUIDv7 — ordering matters here more than anywhere) |
| `event_type` | text | no | `TripConfirmed`, `BookingLapsed`, past tense always |
| `aggregate_schema` / `aggregate_table` | text | no | |
| `aggregate_id` | uuid | no | |
| `subject_trip_id` / `subject_family_id` | uuid | yes | **Denormalised on purpose**: the Timeline is queried by trip and by family, and without these every timeline read becomes a union of joins across thirteen schemas |
| `payload` | jsonb | no | Identity plus what changed — never the whole aggregate |
| `actor_operator_id` | uuid | yes | → operators. Null for system actions. |
| `actor_kind` | actor_kind | no | `operator`, `system`, `auto_policy` |
| `occurred_at` | timestamptz | no | |
| `correlation_id` | uuid | yes | Groups events from one user action |
| `published_at` | timestamptz | yes | Outbox marker for future integrations |

- **PK** `id`
- **Indexes** (`aggregate_schema`, `aggregate_table`, `aggregate_id`, `occurred_at`); **(`subject_trip_id`, `occurred_at DESC`)** and **(`subject_family_id`, `occurred_at DESC`)** — the Timeline; (`event_type`, `occurred_at DESC`) for reporting; `published_at` partial `WHERE published_at IS NULL` (outbox drain)
- **Immutability** Insert-only for every role. No `UPDATE`, no `DELETE`, no exceptions.
- **Soft delete** No · **Partition candidate** by month — **the first table that will need it**
- **Note** This is not event sourcing. Aggregates hold their own current state; this records what happened. It costs one insert per state change and buys the Timeline, the audit narrative, the metrics and the future outbox.

---

**T51 · `audit.record_changes`** — column-level before/after history for regulated and disputed tables.

`id` uuid PK · `schema_name` / `table_name` text · `row_id` uuid · `operation` audit_op (`insert`, `update`, `delete`) · `changed_columns` text[] · `before` jsonb · `after` jsonb · `actor_operator_id` uuid · `occurred_at` timestamptz · `correlation_id` uuid

- **Indexes** (`schema_name`, `table_name`, `row_id`, `occurred_at DESC`); (`actor_operator_id`, `occurred_at DESC`)
- **Coverage** Not every table. Applied by trigger to: families, travellers, documents, rate_contracts, rate_lines, bookings, trips, payment_milestones, operator_roles, sensitive_grants, auto_accept_policies. Everything else is covered by `domain_events`, which is cheaper and more meaningful.
- **Immutability** Insert-only · **Soft delete** No · **Partition candidate** by month

---

**T52 · `audit.security_events`** — authentication, permission and sensitive-access events that are not row changes.

`id` uuid PK · `event_type` text (`role_granted`, `sensitive_access_granted`, `impersonation_started`, `export_performed`) · `actor_operator_id` uuid · `target_operator_id` uuid · `detail` jsonb · `occurred_at` timestamptz · `ip_hash` text

- **Indexes** (`occurred_at DESC`); (`actor_operator_id`, `occurred_at DESC`)
- **Immutability** Insert-only · **Soft delete** No

---

## 3.15 `reporting`

**No base tables.** Materialized views and views only — see §8.4. If a table is ever proposed for this schema, it means something derived is about to become authoritative, and the proposal should be refused.

---

# 4. Relationships

## 4.1 Cardinality map

| Parent | Child | Cardinality | FK behaviour | Boundary |
|---|---|---|---|---|
| `crm.families` | `crm.travellers` | 1 → 0..n | **CASCADE** | Inside aggregate |
| `crm.families` | `crm.contact_identities` | 1 → 0..n | CASCADE | Inside aggregate |
| `crm.families` | `crm.preferences` | 1 → 0..n | CASCADE | Inside aggregate |
| `crm.families` | `crm.families` (merged_into) | 0..1 → 0..n | RESTRICT | Self |
| `crm.families` | `pipeline.enquiries` | 1 → 0..n | RESTRICT | Cross |
| `crm.families` | `planning.trips` | 1 → 0..n | RESTRICT | Cross |
| `pipeline.enquiries` | `planning.trips` | 0..1 → 0..1 | RESTRICT | Cross |
| `planning.trips` | `planning.trip_party` | 1 → 1..n | CASCADE | Inside |
| `planning.trips` | `planning.trip_destinations` | 1 → 1..n | CASCADE | Inside |
| `planning.trips` | `planning.itinerary_days` | 1 → 0..n | CASCADE | Inside |
| `planning.trips` | `planning.itinerary_components` | 1 → 0..n | CASCADE | Inside |
| `planning.itinerary_components` | `planning.component_costs` | 1 → 0..1 | CASCADE | Inside |
| `planning.trips` | `planning.proposals` | 1 → 0..n | **RESTRICT** | Cross — proposals outlive trip edits |
| `planning.trips` | `booking.bookings` | 1 → 0..n | **RESTRICT** | Cross |
| `planning.itinerary_components` | `booking.bookings` | 0..1 → 0..1 | RESTRICT | Cross |
| `booking.bookings` | `booking.booking_travellers` | 1 → 1..n | CASCADE | Inside |
| `crm.travellers` | `booking.booking_travellers` | 1 → 0..n | RESTRICT | Cross |
| `rates.suppliers` | `rates.supplier_contacts` | 1 → 0..n | CASCADE | Inside |
| `rates.suppliers` | `rates.rate_contracts` | 1 → 0..n | RESTRICT | Cross |
| `rates.rate_contracts` | `rates.rate_lines` | 1 → 1..n | **CASCADE** | Inside |
| `rates.rate_contracts` | `rates.rate_contracts` (superseded_by) | 0..1 → 0..1 | RESTRICT | Self |
| `rates.properties` | `rates.property_room_types` | 1 → 0..n | CASCADE | Inside |
| `rates.suppliers` ⟷ `rates.properties` | `rates.supplier_properties` | n ⟷ n | CASCADE both | Join |
| `rates.rate_lines` | `planning.itinerary_components` | 0..1 → 0..n | RESTRICT | Cross — reference only, price is copied |
| `booking.bookings` | `sabre.reservations` | 0..1 → 0..1 | RESTRICT | Cross |
| `sabre.reservations` | `sabre.reservation_segments` | 1 → 0..n | CASCADE | Inside |
| `sabre.reservations` | `sabre.reservation_passengers` | 1 → 0..n | CASCADE | Inside |
| `crm.travellers` | `sabre.reservation_passengers` | 0..1 → 0..n | RESTRICT | Cross, nullable by design |
| `crm.travellers` | `documents.documents` | 1 → 0..n | **RESTRICT** | Cross |
| `documents.documents` | `documents.document_access_log` | 1 → 0..n | RESTRICT | Inside |
| `planning.trips` | `money.payment_schedules` | 1 → 0..1 | RESTRICT | Cross |
| `money.payment_schedules` | `money.payment_milestones` | 1 → 1..n | CASCADE | Inside |
| `money.payment_milestones` | `money.payments` | 1 → 0..n | RESTRICT | Inside, but payments are immutable |
| `ops.task_templates` | `ops.tasks` | 0..1 → 0..n | RESTRICT | Cross |
| `ops.tasks` | six typed subjects | 0..1 → 0..n each | RESTRICT | Cross |
| `intake.inbound_messages` | `intake.message_attachments` | 1 → 0..n | CASCADE | Inside |
| `intake.inbound_messages` | `intake.extractions` | 1 → 0..n | CASCADE | Inside |
| `intake.extractions` | `intake.review_queue_items` | 0..1 → 0..1 | RESTRICT | Cross |
| `intake.review_queue_items` | *(business tables)* | 0..1 → 0..n | **No FK** | Deliberate — see T45 |
| `intake.auto_accept_policies` | `intake.review_queue_items` | 0..1 → 0..n | RESTRICT | Cross |
| `conversations.threads` | `conversations.communications` | 1 → 0..n | RESTRICT | Inside |
| `intake.inbound_messages` | `conversations.communications` | 0..1 → 0..1 | RESTRICT | **Cross-context: the two readings of one arrival** |
| `identity.operators` | *(created_by everywhere)* | 1 → 0..n | RESTRICT | Cross |

## 4.2 The relationships that carry the design

**The spine.** `families → travellers`, `families → trips`, `trips → components`, `trips → bookings`, `travellers → documents`. Everything else hangs off these five.

**The composite-FK chain.** `trip_party` carries both `(trip_id, family_id) → trips(id, family_id)` and `(family_id, traveller_id) → travellers(family_id, id)`. Together they make it impossible — through any code path, including a direct SQL session — to put a traveller from one family on another family's trip. Invariant 6 is not a rule anyone has to remember.

**The one relationship deliberately absent.** `review_queue_items` has no foreign key to the record it targets. It stores schema, table and id as plain values. This is what lets a proposal describe creating a row that does not exist yet, survive its target being deleted, and expire when its target moves. **An FK here would break the trust model** by making a proposal depend on the thing it proposes.

**The relationship that is deliberately weak.** `sabre.reservations.booking_id` is nullable. An unmatched PNR is a normal, visible resting state — not an error, not a foreign key violation, not something to be silently guessed.

**Relationships that do not exist, and why:**

| Not modelled | Reason |
|---|---|
| `trips → documents` | Documents belong to travellers. A trip *raises requirements*, which are derived (§5). |
| `rate_contracts → trips` | Contracts must not know who used them. That is a reporting question answered from `itinerary_components.rate_line_id` and events. |
| `components.booking_id` | The booking points at the component, not the reverse — a component is *realised by* a booking, not composed of one. |
| `enquiries.trip_id` | The Trip references the Enquiry. The Enquiry is upstream and must not be mutated by a downstream context. |
| Anything → `reporting.*` | Reporting is downstream and read-only, always. |

---

# 5. Derived Data — What Must Never Be Stored

Everything in this section is **computed on read** or **materialized as an explicitly rebuildable view**. Nothing here is a column on a base table.

The distinction that matters: derived data may be *materialized*, but it may never be *authoritative*. A materialized view can be dropped and rebuilt from base tables at any moment with no loss. A column cannot.

| Derived concept | Computed from | Why it must not be stored |
|---|---|---|
| **Trip Readiness** | Bookings, documents vs requirements, milestones, blocking tasks | Named in the Domain Model as computed. Cached readiness goes stale silently and is then trusted anyway — the worst failure mode available. A trip that *was* ready is not information anyone wants. |
| **Document Requirements & satisfaction** | `trip_destinations` × `travellers.nationality_code` × `destination_document_policies` × `documents` | The policy is stored; the requirement is not. Storing satisfaction means a policy change in one place silently invalidates rows in another, with nothing to detect the drift. |
| **Timeline** | `events.domain_events` filtered by `subject_trip_id` / `subject_family_id` | Domain Model: the Timeline is a *projection*. The moment code writes a timeline row directly, the narrative and the truth begin to diverge. The denormalised subject columns on `domain_events` exist precisely so no projection table is needed. |
| **Trip margin, total cost, total sell** | `itinerary_components` + `component_costs` | Invariant 9: margin is *always* derived from its parts. A stored margin is a number that can disagree with the itinerary it describes. |
| **Payment outstanding balance** | Milestones − confirmed payments | Invariant 27. Same reason. |
| **Task overdue flag** | `due_at < now()` | Overdue is a *condition*, not a state (Domain Model §6). A boolean would need a cron job to stay true, and would be wrong between runs. |
| **Component booked/delivered status** | Join to `booking.bookings` | The Domain Model is explicit: component state *reflects* its booking. A second copy is a second source of truth. |
| **Family lifetime value, trip count, last-travelled date** | Trips + payments | Aggregates over live data. Storing them creates a nightly job whose failure is invisible. |
| **Supplier performance score** | Bookings, tasks, divergences | Same. Also, the formula will change; stored scores computed under an old formula are worse than none. |
| **Extraction accuracy per kind** | `review_queue_items` decisions | The accuracy number *is* the correction history. Storing a summary invites it to be computed once and quietly never again. |
| **Queue backlog, queue age** | `review_queue_items` where pending | Live operational facts. |
| **Pipeline value, conversion rate, consultant load** | Enquiries, trips, proposals | Dashboard metrics. Materialized views (§8.4), never columns. |
| **Rate coverage %, contract freshness** | Rate contracts vs properties sold | A vision metric, not a business record. |
| **Sabre staleness** | `now() - last_checked_at` | The timestamp is stored; the judgement is not. |
| **"Is this rate quotable?"** | Contract state + validity + supersession | A predicate over the contract row. A `is_quotable` boolean would need updating when a date passes, which nothing triggers. |
| **Trip current state summary / progress %** | Trip state + tasks + bookings | Presentation, not domain. |

**The recurring test:** *if this value can change without any row being written, it must not be a column.* Passport validity changes at midnight. A task becomes overdue at midnight. A contract expires at midnight. None of those are writes, so none of them are columns.

---

# 6. Audit Strategy

## 6.1 The four layers

Audit is not one mechanism. Four layers answer four different questions, and conflating them produces a system that answers none of them well.

| Layer | Question it answers | Where |
|---|---|---|
| **Row stamps** | Who last touched this row, and when? | `created_at/by`, `updated_at/by` on every mutable table |
| **Domain events** | What happened in the business? | `events.domain_events` |
| **Change history** | What exactly changed on this sensitive row? | `audit.record_changes` |
| **Provenance** | Where did this fact come from, and who authorised it? | `source_review_item_id` + the review item's own record |

## 6.2 Row stamps

| Column | Type | Rule |
|---|---|---|
| `created_at` | timestamptz NOT NULL DEFAULT now() | Never updated. |
| `created_by` | uuid NULL → operators | Null only for system-created rows (extraction workers, scheduled jobs), where `actor_kind` on the corresponding event says which. |
| `updated_at` | timestamptz NULL | Set by trigger on every UPDATE. Null means never modified since creation — which is itself useful. |
| `updated_by` | uuid NULL → operators | Set by trigger from the session's operator context. |

`created_by` and `updated_by` are `RESTRICT` foreign keys to `identity.operators`, which is why operators are never deleted (T1). Attribution outliving employment is a requirement, not a side effect.

Append-only tables carry `created_at`/`created_by` only. An `updated_at` on an immutable table is a lie waiting to be told.

## 6.3 Domain events

One row in `events.domain_events` per meaningful state change, **written in the same transaction as the change**. Not after, not by a queue, not best-effort — the event and the state change either both happen or neither does.

- Named in the past tense, from the Domain Model's event list.
- Payload carries identity and what changed, never the whole aggregate.
- `correlation_id` groups everything caused by one user action, so "what did accepting that review item actually do?" is one query.
- `actor_kind` distinguishes `operator`, `system` and `auto_policy` — the last is how auto-accepted changes stay visibly machine-originated forever.

## 6.4 Change history

`audit.record_changes` is a trigger-based before/after log on the eleven tables listed in T51: the regulated ones (documents, families, travellers), the commercial ones (rate contracts and lines, bookings, milestones), and the security ones (roles, grants, auto-accept policies).

It is deliberately *not* on every table. Universal row-level auditing doubles write volume and buries the rows anyone would actually read. Everything else is covered by domain events, which say what happened rather than which bytes moved.

## 6.5 Review provenance

Every record that intake can create or change carries `source_review_item_id`. That single nullable column chains to the whole story:

```
business row
  └─ source_review_item_id → review_queue_items
                               ├─ decided_by            (which human)
                               ├─ auto_accept_policy_id (or which named policy)
                               ├─ proposed_change / corrected_change
                               └─ extraction_id → extractions
                                                    ├─ model_name / model_version
                                                    └─ message_id → inbound_messages (the original email)
```

**Provenance is a reference, not a copy** (§1.7). Duplicating model version and approver onto every business row would be four columns of drift waiting to happen, and the chain is one join deep.

This chain is what makes the vision's central question answerable: *which extraction kinds have earned wider authority, and who decided that?* Without it, "should we let the AI do this alone?" is a matter of nerve rather than evidence.

## 6.6 Immutable history

Enforced by trigger, not convention:

| Table | Rule |
|---|---|
| `events.domain_events`, `audit.*` | Insert only. No update, no delete, for any role. |
| `intake.inbound_messages`, `intake.extractions` | Content immutable; `state` may advance. |
| `planning.proposals` | Snapshot immutable once `state <> 'draft'` (invariant 11). |
| `money.payments` | Immutable once confirmed; reversals are new rows (invariant 26). |
| `documents.document_access_log` | Insert only, including for administrators. |
| `rates.rate_contracts` | Commercial columns immutable once active; amendments supersede (invariant 19). |

**Nothing in the system has a "delete history" path.** Purging a document destroys the file and keeps the tombstone; that is the closest thing to deletion the design permits, and it exists only to satisfy retention obligations.

---

# 7. RLS Strategy

## 7.1 The Supabase constraint that shapes everything

Every signed-in operator connects as the same PostgreSQL role (`authenticated`). Column-level `GRANT`s therefore **cannot** distinguish a consultant from an operations executive — grants apply to the role, and both are the same role.

**Row Level Security is the only per-user mechanism available, and RLS is row-level.** Every access decision in this design is therefore expressed as *which rows*, never *which columns*. Where a genuine column-level need exists — supplier cost, and therefore margin — the sensitive columns are split into their own table (`planning.component_costs`, T20) so that a row policy can protect them.

This is not a workaround. It is the honest consequence of the platform, and designing around it now avoids discovering it after the schema is built.

## 7.2 Foundations

1. **RLS enabled on every table in every schema.** Including lookups. Including `reference`. No exceptions "because it's just reference data" — an exception list is a thing people add to.
2. **Default deny.** No permissive catch-all policy exists anywhere.
3. **Policies call helper functions, never inline logic.** A small set of `STABLE SECURITY DEFINER` functions in a dedicated `security` schema — `current_operator_id()`, `has_role(role)`, `has_sensitive_grant(tier)`, `owns_family(uuid)`, `can_see_trip(uuid)`. Three reasons: policies stay readable, the functions can be tested independently, and **changing the access model later is a change to a handful of functions rather than a hundred policies**. This is also what makes future multi-tenancy a small change (§10).
4. **Avoid recursive RLS.** Helper functions are `SECURITY DEFINER` so that checking "does this operator have this role?" does not itself trigger RLS on `identity.operator_roles` and recurse.
5. **The intake worker uses the service role**, which bypasses RLS — and this is exactly why intake writes only to `intake` tables. A bypassing role that could write to `crm` would defeat the entire trust model. **The service role's write surface is the schema boundary.**
6. **Every policy filters `deleted_at IS NULL`** on soft-deletable tables, so soft-deleted rows are invisible without application cooperation.

## 7.3 Access model by role

### Travel Consultant
- **Families and travellers** — full read and write on families they own (`relationship_owner_id = current_operator_id()`); **read-only on all other families**. Visibility across the agency is a deliberate feature: the Vision states this business loses far more to siloing than to snooping.
- **Trips** — write on trips they own; read on all.
- **Enquiries** — write on their own; read on all.
- **Rates** — read on everything quotable; **no write on contracts or lines.**
- **Component costs (T20)** — read only for trips they own. A consultant sees margin on their own book, and not on a colleague's.
- **Bookings** — write on their own trips before confirmation; read after.
- **Documents** — metadata for travellers on their trips; **file access requires the `identity_documents` grant.**
- **Review queue** — read all; may decide items whose target belongs to their families.
- **Tasks** — read all; write on assigned or own-trip tasks.

### Operations Executive
- **Trips** — read all; write on trips in delivery states (`confirmed`, `in_progress`).
- **Bookings** — full write on all live trips. This is the job.
- **Documents** — **full access to file and detail**, because collection and verification is the core function. The control is `document_access_log` (T36), which is insert-only and covers everyone. The Domain Model is explicit that the control here is the audit trail, not a lock.
- **Component costs** — **no access.** This is the single most important negative permission in the design, and the reason T20 exists as a separate table.
- **Booking cost** — visible, because paying suppliers requires it. Supplier cost on a booking and per-component margin are different questions.
- **Money** — full write on schedules, milestones and payments.
- **Review queue** — full decision rights on all items except those restricted by risk class.
- **Sabre** — full read and resolution rights.

### Founder
- **Read on everything**, including `component_costs`, `reporting`, all commercials.
- **Write** limited to ownership reassignment (`families.relationship_owner_id`, `trips.owner_id`) and auto-accept policy changes.
- **No blanket write.** A read-mostly role that can nonetheless edit anything is the fastest route to a dashboard nobody trusts, because nobody can say whether a number was edited.
- Document *file* access still requires an explicit `identity_documents` grant. Seniority is not a grant.

### Administrator
- **Full write** on `identity`, `reference`, `rates` master data, `ops.task_templates`.
- **No automatic access to `documents` file content or `component_costs`** — Domain Model rule 49, enforced by policy: administrative authority does not confer data access. An admin who needs a passport gets an explicit, logged, revocable grant like anyone else.
- Read on `audit` and `events`.

### Future Client *(not built; the model must accommodate it)*
Access decomposes into the same three dimensions already in use, which is why no restructuring is needed:

- **Role** — a new `client` role.
- **Assignment** — linked to a Traveller, and through it a Family. `can_see_trip()` gains one branch: a client sees trips whose `family_id` matches their traveller's family.
- **Sensitivity** — clients never hold the `commercials` grant, so `component_costs` is unreachable by construction. They hold `identity_documents` only for their own family's travellers.

The concrete work when that day comes: add the role, add one branch to two helper functions, and add `client_user_id` to a link table. **No base table changes.** That is the entire point of routing every policy through helper functions.

## 7.4 Table-class summary

| Class | Read | Write |
|---|---|---|
| `reference.*` | All operators | Administrator |
| `crm.*` | All operators | Owner consultant, Administrator |
| `planning.trips`, components | All operators | Owner consultant; Operations when confirmed |
| `planning.component_costs` | Owner consultant, Founder | Owner consultant |
| `rates.*` contracts and lines | All operators | Administrator only |
| `booking.*` | All operators | Owner consultant, Operations |
| `documents.*` metadata | All operators | Operations |
| `documents.*` files and detail | `identity_documents` grant holders | Operations with grant |
| `money.*` | Operations, Founder | Operations |
| `intake.*` | All operators | Service role (worker) + decision columns by operators |
| `conversations.*` | All operators | Author |
| `events`, `audit` | Founder, Administrator | **Nobody** — insert only, via trigger |
| `reporting.*` | By view; commercial views gated on the `commercials` grant | Nobody |

## 7.5 Known gaps to close before launch

- **Storage bucket policies must mirror table RLS.** A document row protected by RLS whose file sits in a public bucket is not protected. Supabase Storage policies are a separate surface and a separate review.
- **`intake.inbound_messages` holds client correspondence**, which is a wider privacy surface than the passports everyone thinks about first. It needs its own retention policy and its own read restriction — probably narrower than "all operators".
- **Service-role key handling** is the single highest-value credential in the system, since it bypasses RLS entirely.

---

# 8. Performance

Sizing first, because it determines how much of this is warranted. At the Vision's stated scale — 10–20 operators, low thousands of trips per year — every table except four stays under a million rows for years. **The four that grow without bound are `events.domain_events`, `intake.inbound_messages`, `audit.record_changes` and `documents.document_access_log`.** Effort belongs there, and almost nowhere else.

## 8.1 Index strategy

Indexes exist for known query shapes, not for speculation. The queries that actually run constantly:

| Query | Index |
|---|---|
| **Rahul's daily task list** | `ops.tasks (assignee_id, state, due_at) WHERE state IN ('open','in_progress')` |
| **The review queue** | `intake.review_queue_items (state, risk_class, created_at) WHERE state = 'pending'` |
| **Rate search** — destination + dates + category | `rates.rate_lines USING gist (property_id, season)`; `rates.properties (destination_id)`; contract state partial index |
| **Consultant's book** | `pipeline.enquiries (owner_id, state)`; `planning.trips (owner_id, state)` |
| **Passport expiry sweep** | `documents.documents (expires_on) WHERE state IN ('valid','expiring')` |
| **Hold expiry sweep** | `booking.bookings (hold_expires_at) WHERE state = 'held'` |
| **Payment chasing** | `money.payment_milestones (due_on) WHERE state IN ('pending','due','part_paid')` |
| **Sabre work queue** | `sabre.reservations (sync_state) WHERE sync_state IN ('unmatched','diverged')` |
| **Timeline for a trip** | `events.domain_events (subject_trip_id, occurred_at DESC)` |
| **Inbound sender matching** | `crm.contact_identities (channel, identifier)`; `rates.supplier_contacts (channel, identifier)` |
| **Accepted-but-not-applied defects** | `intake.review_queue_items (state) WHERE state = 'accepted'` |

**Partial indexes are the dominant pattern**, because almost every operational query is about the small live subset of a mostly-historical table. A partial index on open tasks stays small forever, however many tasks are completed.

**Every foreign key used for filtering gets an index**; PostgreSQL does not create them automatically, and the omission surfaces as slow cascades and lock contention rather than slow selects, which makes it hard to diagnose.

**Do not index** low-cardinality booleans alone, columns only ever read by id, or `created_at` on tables nobody sorts by.

## 8.2 Search strategy

Three different problems, three different tools — using one tool for all three is the usual mistake:

1. **Structured filtering** (destination, dates, category, price band) — ordinary B-tree and GiST indexes. This is the rate search, and it is the one that must be under 30 seconds end-to-end per the Vision's metric. It will be under 50 ms.
2. **Full-text search** (properties, families, messages, notes) — `tsvector` **generated columns** with GIN indexes. Generated, not trigger-maintained, so they cannot drift. `simple` plus `english` configuration; property and family names are often non-English, so `unaccent` is applied.
3. **Fuzzy name matching** (duplicate travellers, sender matching, "the Mehtas" vs "Mehta family") — `pg_trgm` with GIN trigram indexes on names. **This directly serves the Domain Model's named risk of traveller duplication**, and it is the mechanism behind intake's candidate matching.

Deliberately deferred: `pgvector` and semantic search. Supabase supports it, it will be wanted for the AI phase, and adding an embeddings table later touches nothing here (§10).

## 8.3 Partitioning candidates

**None at launch.** Partitioning a table with 50,000 rows costs planning time and query complexity and buys nothing.

Ordered by when they will actually need it:

| Table | Partition by | Trigger point |
|---|---|---|
| `events.domain_events` | RANGE on `occurred_at`, monthly | ~10M rows, or when index maintenance becomes visible. **The first to need it.** |
| `intake.inbound_messages` | RANGE on `received_at`, monthly | ~5M rows, or when retention requires dropping old partitions — which is the real driver, since dropping a partition beats deleting rows. |
| `audit.record_changes` | RANGE on `occurred_at`, monthly | Alongside events |
| `documents.document_access_log` | RANGE on `accessed_at`, monthly | Only if access volume surprises us |
| `conversations.communications` | RANGE on `occurred_at`, quarterly | Well after WhatsApp arrives |

The design is partition-*ready*: each has a natural time column, no unique constraint that would need to include the partition key awkwardly, and no foreign key pointing *into* it that would complicate attachment.

## 8.4 Materialized views

All in `reporting`, all refreshed `CONCURRENTLY` on a `pg_cron` schedule, all rebuildable from scratch. Each carries a `refreshed_at` column so a stale dashboard says so instead of lying quietly — the Domain Model's "a wrong number kills the dashboard" risk, addressed at the schema level.

| View | Contents | Refresh |
|---|---|---|
| `reporting.mv_pipeline_summary` | Enquiries by state, owner, value, age | 15 min |
| `reporting.mv_trip_commercials` | Cost, sell, margin per trip. **Gated on the `commercials` grant.** | 15 min |
| `reporting.mv_consultant_load` | Live trips, open tasks, pipeline value per consultant | Hourly |
| `reporting.mv_queue_health` | Pending count, age distribution, per-kind acceptance and correction rates | 5 min — the AI-readiness metric, and the one the founder reads monthly |
| `reporting.mv_rate_coverage` | Properties sold vs properties with in-date contracts | Daily |
| `reporting.mv_trips_at_risk` | Blocking tasks overdue, documents expiring, divergences, milestones overdue | 15 min |
| `reporting.mv_document_readiness` | Trips departing in 30 days vs requirement satisfaction | Hourly |

Plain (non-materialized) views for anything that must be live: trip readiness, outstanding balance, current margin on a single trip. **Readiness is never materialized** — §5.

## 8.5 Caching candidates

| Layer | What | Why |
|---|---|---|
| PostgreSQL | Materialized views above | The only cache that is also a database object, and therefore the only one that cannot silently diverge |
| Application | `reference.*`, `ops.task_templates`, `pipeline.loss_reasons` | Small, slow-moving, read constantly |
| Application, short TTL | Rate search results within a session | Consultants re-run near-identical searches while designing |
| **Never cached** | Trip readiness, document validity, review queue contents, permission checks | Every one is a correctness question. A stale permission or a stale passport expiry is worse than a slow one. |

## 8.6 Operational notes

- **Connection pooling** via Supabase's pooler in transaction mode. Server Components open many short connections; this is the usual first production surprise.
- **`REFRESH MATERIALIZED VIEW CONCURRENTLY` requires a unique index** on each view. Easy to forget, and it fails at 3 a.m. rather than in review.
- **`pg_stat_statements` on from day one.** Retrofitting query visibility after a performance problem wastes the days when it matters most.
- **Autovacuum tuning** on `events.domain_events` and `intake.inbound_messages` — high insert, no update, so the defaults are wrong in the direction of never vacuuming until it hurts.

---

# 9. Migration Order

Dependency-ordered. Each phase depends only on earlier phases, so each is independently deployable and independently testable.

### Phase 0 — Foundation *(no tables)*
Extensions (`pgcrypto`, `pg_trgm`, `unaccent`, `citext`, `pg_cron`), schemas, all enum types, the `security` helper functions, and the shared trigger functions for `updated_at`, audit and immutability. **Enums and helper functions come first because everything after references them**, and retrofitting an enum onto a populated text column is a migration nobody enjoys.

### Phase 1 — Identity and reference
**T1** operators → **T2** operator_roles → **T3** sensitive_grants → **T4** countries → **T5** destinations → **T6** destination_document_policies

*Why first:* every table's `created_by` points at operators, and every RLS policy calls a function that reads roles and grants. Nothing can be secured before this exists.

### Phase 2 — Events and audit
**T50** domain_events → **T51** record_changes → **T52** security_events

*Why this early:* audit triggers are attached as each subsequent table is created. Adding the log after the tables means either backfilling or accepting a permanent gap at the start of the system's life.

### Phase 3 — CRM
**T7** families → **T8** travellers → **T9** contact_identities → **T10** preferences → **T11** family_links → **T12** family_merges

*Note:* families and travellers are mutually referential (`principal_traveller_id`), so the FK is `DEFERRABLE INITIALLY DEFERRED` and added after both tables exist.

### Phase 4 — Supply and rates
**T22** suppliers → **T23** supplier_contacts → **T24** properties → **T25** property_room_types → **T26** supplier_properties → **T27** rate_contracts → **T28** rate_lines

*Why before planning:* components reference rate lines. Also the Vision's sequencing — rates are Phase 2 of the build and the module that proves the pipeline's value fastest.

### Phase 5 — Pipeline and planning
**T14** loss_reasons → **T13** enquiries → **T15** trips → **T16** trip_party → **T17** trip_destinations → **T18** itinerary_days → **T19** itinerary_components → **T20** component_costs → **T21** proposals

*Note:* `trips.accepted_proposal_id` is deferrable and added after T21. `trip_party` needs the composite unique keys on both `trips` and `travellers` to exist first — those must not be forgotten in Phases 3 and 5, because they are what make invariant 6 enforceable.

### Phase 6 — Booking and Sabre
**T29** bookings → **T30** booking_travellers → **T31** sabre.reservations → **T32** segments → **T33** passengers → **T34** sync_runs

*Why Sabre here:* reservations reference bookings and travellers. **This phase can slip without blocking anything else** — the Vision's stated fallback if Sabre access proves harder than assumed. Bookings work standalone; the mirror is additive.

### Phase 7 — Documents
**T35** documents → **T36** document_access_log

*Why after CRM:* documents belong to travellers. *Why its own phase:* the strictest RLS and the Storage bucket policies land together, and that pairing deserves its own review rather than being buried in a larger deployment.

### Phase 8 — Operations and money
**T37** task_templates → **T38** tasks → **T39** payment_schedules → **T40** payment_milestones → **T41** payments

*Why here:* tasks reference six subjects across five earlier phases. This is the last phase that can be built, which is fine — it is also the phase that turns confirmation into an operational plan.

### Phase 9 — Intake and conversations
**T42** inbound_messages → **T43** message_attachments → **T44** extractions → **T46** auto_accept_policies → **T45** review_queue_items → **T47** threads → **T48** communications → **T49** notes

*Why last, despite being first in the product:* review items carry `target_schema`/`target_table` values that must resolve to real tables, and business tables carry `source_review_item_id` pointing back. Building intake last means those pointers are added as a **final alteration pass** rather than as forward references — the one place where the build order deliberately inverts the product order.

*Note:* T46 precedes T45 because the review item's `auto_accept_policy_id` FK and its risk-class check constraint both depend on it.

### Phase 10 — Reporting
Materialized views, their unique indexes, `pg_cron` refresh schedules.

*Why last:* everything it reads must exist. Nothing depends on it, so it can also be rebuilt or redesigned freely — which is the point of a downstream context.

### Cross-cutting, applied per phase
RLS enablement and policies ship **with each table, in the same migration** — never as a later pass. A table that exists without RLS for even one deployment is a table someone will query without it.

---

# 10. Future Expansion

Each of these was designed for, and none requires changing a core table.

## 10.1 WhatsApp

**Already accommodated.** Three things carry it:

1. `intake.inbound_messages.channel` is an enum with `whatsapp` as a value. The classify → extract → propose → review path is channel-agnostic already.
2. `crm.contact_identities` and `rates.supplier_contacts` already key on `(channel, identifier)`. A WhatsApp number matches a family through the same unique index an email address does today.
3. `conversations.communications.channel` already distinguishes channels, and the table already models outbound with an approval gate.

**The change:** an ingestion worker for the WhatsApp Business API, and one new enum value that is already present. **No schema change to any business table.** This is what the Domain Model meant by "adding a channel, not a second pipeline," expressed in DDL.

*The one addition likely needed:* an outbound delivery-status table if WhatsApp receipts matter operationally — additive, isolated, touching nothing.

## 10.2 AI agents

**Already accommodated**, through the generalisation the Domain Model identified as the key evolutionary insight.

`intake.review_queue_items` describes a *proposed change*, not an extracted record: `kind`, `proposed_change` (jsonb), `target_*`, `risk_class`, decision columns. An agent proposing "hold this suite" or "send this reply" is **a new `review_item_kind` enum value and a new payload shape** — no new table, no new lifecycle, no new approval mechanism.

The safety properties come along unchanged:
- Rule 32 is a check constraint: no decided item without a human or a named policy.
- Rule 33 is a check constraint: `risk_class = 'restricted'` can never be auto-accepted.
- Rule 37 is a check constraint on `communications`: a machine-drafted message cannot reach `sent` without an approver — **already enforced, years before the feature exists.**
- `auto_accept_policies` already governs where machine authority may widen, with an owner and an audit trail.

**Additive when the time comes:** an `ai.embeddings` table (`subject_kind`, `subject_id`, `vector`, `model_version`) with a `pgvector` index, for semantic search over families, trips and properties. It references business rows by id and nothing references it — a pure leaf, droppable without consequence.

## 10.3 Sabre

The `sabre` schema is a self-contained anti-corruption layer. Deepening the integration — schedule change ingestion, ticketing automation, queue monitoring — adds tables *inside* `sabre` and changes nothing outside it.

**Adding a second GDS or a supplier API** means a new schema with the same shape: a mirror table with `sync_state`, a segment or item table, a mapping table with nullable business references, and a sync-run log. The pattern is established once and copied. `booking.bookings` is unaware of how many external systems exist, which is the entire purpose of the boundary.

## 10.4 Payments

`money.payments` records that money moved. It does not move money, and the schema has no gateway concepts in it — which is what makes adding one clean.

**When client payment collection arrives:** a `money.payment_intents` table (provider, provider reference, status, amount, milestone link) sits *upstream* of `payments`. A successful intent produces a `payments` row exactly as a bank transfer does today. Reconciliation, refunds and chargebacks all attach to intents, leaving the payment record — the immutable financial fact — untouched.

Multi-currency settlement, if it becomes real, adds an `money.fx_rates` table and an explicit rate-and-date on conversions. Deliberately absent today (§1.9): implicit conversion is how financial systems become quietly wrong.

## 10.5 Multi-tenancy

Not built (Vision non-goal N7). The path is kept cheap by one decision: **every RLS policy calls a helper function rather than inlining logic.**

If a second agency ever becomes real:

1. Add `tenant_id uuid NOT NULL` to root tables only — families, trips, suppliers, properties, operators. Child tables inherit tenancy through their parent and need no column.
2. Backfill a single tenant value.
3. Add `current_tenant_id()` to the `security` schema and one clause to each helper function.
4. Extend the composite indexes that matter to lead with `tenant_id`.

**Policies themselves do not change**, because they already delegate. This is roughly a day of work on a schema of this size, and it is only that cheap because policies were centralised from the start.

*The alternative* — schema-per-tenant — is viable at very low tenant counts and worse beyond a handful. Neither should be chosen until there is a real second tenant, since the choice depends entirely on whether tenants share suppliers and rates.

## 10.6 Client access

Covered in §7.3. Add a `client` role, link a client user to a traveller, add one branch to two helper functions. No base table changes, because the access model was built on three independent dimensions — role, assignment, sensitivity — rather than on the assumption that every user is staff.

---

## Appendix — Open Questions for CTO Review

1. **UUIDv7 availability.** Which PostgreSQL version does the target Supabase project run, and is `uuidv7()` available natively? Affects index behaviour on the four high-growth tables, and is easiest to decide before the first migration.
2. **`component_costs` split (T20).** Confirms a real ergonomic cost — every price edit touches two tables — in exchange for the only workable per-user cost visibility on Supabase. Worth an explicit decision rather than an inherited one.
3. **Correspondence retention.** How long are `intake.inbound_messages` bodies kept? This is a larger privacy surface than passports and currently has no stated policy.
4. **`document_access_log` volume.** Logging every metadata read may be noisy. Should `metadata_only` reads be sampled, or is complete coverage the requirement?
5. **Rate line granularity.** Does the business price per room type per season per occupancy, or are there contracts with per-night or per-person-per-night structures the current shape cannot express? Worth checking against three real contracts before building.
6. **Blackout dates as `daterange[]`.** Simple, but not indexable for exclusion queries. If "find rates excluding blackouts" becomes a hot search, this wants its own table.
7. **Trip commercials currency.** Do trips ever mix currencies across components? §1.9 forbids cross-currency arithmetic in the database, so a mixed-currency trip needs a stated presentation rule.
8. **Task subject columns (T38).** Six typed nullable FKs give real referential integrity; a polymorphic `subject_id` would be narrower but unenforced. Confirm the trade is acceptable, since it sets a precedent for `conversations.threads` and `notes` too.

---

# Appendix B — Domain Concept → Database Representation

Exhaustive mapping from [02-domain-model.md](02-domain-model.md) to this schema. Every concept in the Domain Model appears here exactly once. Four outcomes are possible:

- **Table** — a persistent business concept with its own rows.
- **Columns** — a value object, stored inline on its owner. Never its own table.
- **Computed** — derived on read. Must never be a column (§5).
- **Projection** — materialized or queried from `events.domain_events`. Rebuildable, never authoritative.

## B.1 Aggregate roots

| Domain concept | Database representation | Notes |
|---|---|---|
| Family | **`crm.families`** (T7) | Aggregate root |
| Supplier | **`rates.suppliers`** (T22) | |
| Property | **`rates.properties`** (T24) | |
| Enquiry | **`pipeline.enquiries`** (T13) | |
| Trip | **`planning.trips`** (T15) | Core of the model |
| Proposal | **`planning.proposals`** (T21) | Immutable snapshot; `jsonb` + frozen totals |
| Rate Contract | **`rates.rate_contracts`** (T27) | Strategic asset |
| Booking | **`booking.bookings`** (T29) | Outside `planning` by design |
| Sabre Reservation | **`sabre.reservations`** (T31) | Mirror only, never master |
| Payment Schedule | **`money.payment_schedules`** (T39) | |
| Task | **`ops.tasks`** (T38) | |
| Document | **`documents.documents`** (T35) | |
| Inbound Message | **`intake.inbound_messages`** (T42) | Immutable |
| Review Queue Item | **`intake.review_queue_items`** (T45) | The trust layer |
| Communication | **`conversations.communications`** (T48) | Append-only |
| Operator | **`identity.operators`** (T1) | Linked to `auth.users` |

## B.2 Entities inside an aggregate

| Domain concept | Database representation | Parent aggregate |
|---|---|---|
| Traveller | **`crm.travellers`** (T8) | Family |
| Itinerary | **`planning.itinerary_days`** (T18) | Trip — the Itinerary has no row of its own; it *is* the ordered set of days |
| Itinerary Day | **`planning.itinerary_days`** (T18) | Trip |
| Itinerary Component | **`planning.itinerary_components`** (T19) | Trip |
| Rate Line | **`rates.rate_lines`** (T28) | Rate Contract |
| Extraction | **`intake.extractions`** (T44) | Inbound Message |
| Payment Milestone | **`money.payment_milestones`** (T40) | Payment Schedule |
| Payment | **`money.payments`** (T41) | Payment Schedule — immutable once confirmed |
| Passport | **`documents.documents`** with `kind = 'passport'` (T35) | *Not a separate table.* Typed detail in `detail` jsonb, with `document_number`, `issued_on`, `expires_on`, `issuing_country` promoted to columns for indexing and uniqueness |
| Note *(internal remark)* | **`conversations.notes`** (T49) | Distinct from Communication — never leaves the building |
| Thread | **`conversations.threads`** (T47) | |

## B.3 Supporting tables with no single Domain Model concept

These exist because a relational model needs them; each still maps to something a consultant would recognise.

| Business meaning | Database representation |
|---|---|
| Who is on this trip | **`planning.trip_party`** (T16) — carries the composite FKs enforcing invariant 6 |
| Where this trip goes | **`planning.trip_destinations`** (T17) |
| Who this booking names | **`booking.booking_travellers`** (T30) |
| Which suppliers sell this property | **`rates.supplier_properties`** (T26) |
| Room categories a property sells | **`rates.property_room_types`** (T25) |
| Files attached to an email | **`intake.message_attachments`** (T43) |
| Flight segments as Sabre reports them | **`sabre.reservation_segments`** (T32) |
| Sabre passengers, and our mapping to travellers | **`sabre.reservation_passengers`** (T33) |
| Sync attempt history | **`sabre.sync_runs`** (T34) |
| Roles held by an operator | **`identity.operator_roles`** (T2) |
| Sensitivity-tier grants | **`identity.sensitive_grants`** (T3) |
| Structured preferences | **`crm.preferences`** (T10) |
| How to recognise a family on a channel | **`crm.contact_identities`** (T9) |
| Families that travel together | **`crm.family_links`** (T11) |
| Record of a family merge | **`crm.family_merges`** (T12) |
| Supplier contacts and channel identifiers | **`rates.supplier_contacts`** (T23) |
| Supplier cost per component | **`planning.component_costs`** (T20) — split for RLS, §7.1 |
| Why enquiries are lost | **`pipeline.loss_reasons`** (T14) |
| Rules that generate tasks | **`ops.task_templates`** (T37) |
| Where machine authority is permitted | **`intake.auto_accept_policies`** (T46) |
| Countries | **`reference.countries`** (T4) |
| Destinations | **`reference.destinations`** (T5) |

## B.4 Value objects — columns, never tables

| Domain concept | Database representation |
|---|---|
| Money | Column pair: `numeric(14,2)` amount + `char(3)` currency, always together, always both null or both set |
| Date Range | `daterange` columns — `trips.travel_span`, `rate_lines.season`, `rate_contracts.validity`, `itinerary_components.stay_span` |
| Party | `planning.trip_party` rows plus `party_adults` / `party_children` on enquiries; occupancy shape on `rate_lines.occupancy_basis` |
| Confidence | `intake.extractions.field_confidence` (jsonb, per field) and `overall_confidence` (numeric) |
| Sync State | `sabre.reservations.sync_state` enum column |
| **Provenance** | **`source_review_item_id` FK** on every intake-writable table, chaining to review item → extraction → message. A reference, not a copy (§6.5) |
| Commercials | `itinerary_components.unit_sell` + `planning.component_costs.unit_cost`. **Margin itself is computed** — see B.5 |
| Destination Policy | **`reference.destination_document_policies`** (T6) — the *policy* is stored; the requirement it produces is not |
| Family role, tier, strength, risk class, all lifecycle states | Native PostgreSQL enum types (§1.8) |

## B.5 Computed — must never be stored

| Domain concept | How it is obtained |
|---|---|
| **Trip Readiness** | **Computed** — live view over bookings, documents vs requirements, milestones, blocking tasks. Never materialized. |
| **Document Requirement** | **Computed** — `trip_destinations` × `travellers.nationality_code` × `destination_document_policies` × `documents` |
| Document Requirement satisfaction | **Computed** — same join, evaluated against travel dates |
| Trip margin / total cost / total sell | **Computed** — invariant 9 |
| Payment outstanding balance | **Computed** — milestones minus confirmed payments, invariant 27 |
| Task *overdue* | **Computed** — `due_at < now()`; a condition, not a state |
| Component *booked* / *delivered* status | **Computed** — join to `booking.bookings`; the component only stores `selection_state` |
| Rate *quotable* | **Computed** — contract state + validity + not superseded |
| Sabre *stale* | **Computed** — `now() - last_checked_at` |
| Family lifetime value / trip count / last travelled | **Computed** |
| Supplier performance | **Computed** — from bookings, tasks and divergences |
| Extraction accuracy per kind | **Computed** — from `review_queue_items` decisions |
| Queue backlog and age | **Computed** |
| Minor status | **Computed** — from `date_of_birth`; `family_role = 'minor'` is the business assertion, age is the fact |

## B.6 Projections — derived, rebuildable, never authoritative

| Domain concept | Database representation |
|---|---|
| **Timeline Event** | **Projection** — queried from `events.domain_events` (T50) filtered by `subject_trip_id` / `subject_family_id`. No projection table exists; the denormalised subject columns are what make one unnecessary. |
| Domain Event *(the fact itself)* | **`events.domain_events`** (T50) — this one *is* a table: an event is a persisted fact, not a derivation |
| Pipeline value / conversion | `reporting.mv_pipeline_summary` |
| Trip commercials rollup | `reporting.mv_trip_commercials` — gated on the `commercials` grant |
| Consultant load | `reporting.mv_consultant_load` |
| Queue health / AI accuracy | `reporting.mv_queue_health` |
| Rate coverage | `reporting.mv_rate_coverage` |
| Trips at risk | `reporting.mv_trips_at_risk` |
| Document readiness across trips | `reporting.mv_document_readiness` |
| Dashboard metrics *(all)* | **Projection** — materialized views only, each with `refreshed_at` |

## B.7 Concepts with no direct representation, deliberately

| Domain concept | Why |
|---|---|
| Aggregate boundary | Expressed as FK behaviour: `CASCADE` inside, `RESTRICT` across (§1.2). Not a row. |
| Bounded context | Expressed as a PostgreSQL schema (§2). Not a row. |
| Bounded-context relationship *(Customer/Supplier, ACL, Conformist)* | Expressed as schema boundaries plus the service role's restricted write surface (§7.2). Not a row. |
| Business invariant | Expressed as check constraints, composite FKs, partial unique indexes and exclusion constraints — with rules 32, 33 and 37 enforced as check constraints specifically because they must never become negotiable. |
| Policy *(eventually consistent, cross-aggregate)* | Expressed as `ops.task_templates` rows plus event-driven task generation. Not enforced by the database, by design — policies are not invariants. |
| Ubiquitous Language | Expressed as table, column and enum naming throughout. `Booking` and `Reservation` are separate schemas precisely so the distinction cannot erode. |

## B.8 Audit and infrastructure

| Business meaning | Database representation |
|---|---|
| Who last touched this row | `created_at` / `created_by` / `updated_at` / `updated_by` — standard block (§6.2) |
| What changed on a sensitive row | **`audit.record_changes`** (T51) — eleven tables only |
| Security and permission events | **`audit.security_events`** (T52) |
| Every read of a document | **`documents.document_access_log`** (T36) — insert-only for every role |
| Business facts, timeline source, future outbox | **`events.domain_events`** (T50) |
