# PureLuxe Studio — Implementation Plan, Sprint 1

**For:** CTO review, then execution by Claude Code
**Date:** 25 July 2026
**Follows:** [01-product-vision.md](01-product-vision.md) v1.0 · [02-domain-model.md](02-domain-model.md) v1.0 · [03-database-design.md](03-database-design.md) v1.0
**Contains:** No application code, no SQL, no components, no migrations. This is the blueprint those are written from.

---

## Preface — the three source documents are binding

Everything below implements what those documents already decided. Where this plan makes a choice they did not cover, it is a *delivery* choice — how to wire Supabase Auth, where a file lives, what a service returns — never a domain choice.

Three standing rules for the whole build:

1. **The ubiquitous language from Domain Model §2 is enforced in code.** `family`, never `customer`. `operator`, never `user`. `traveller`, never `passenger`. This applies to file names, variables, types, routes and copy.
2. **Nothing derived is ever stored.** Database Design §5 is a list of things that must not become columns, fields on a type, or cached values in React state that outlive a render.
3. **Schema boundaries are context boundaries.** Code that touches `crm` lives in the CRM feature. A component that reaches across contexts is a design defect, not a shortcut.

---

# 1. Sprint Goal

**After Sprint 1, a real consultant can sign in and create a real family with real travellers, and that data is correctly owned, audited and protected.**

Deliberately small. The point of Sprint 1 is not features — it is to prove the whole vertical works end to end on the actual hosted Supabase project: auth → RLS → server component → service layer → validated mutation → domain event → audit trail. Every later sprint copies this shape. If it is wrong, it is wrong forty times.

### What exists at the end of Sprint 1

| Capability | Detail |
|---|---|
| **Authentication** | An invited operator signs in with email. Public signup is disabled. Sessions persist and refresh. Signing out works. |
| **Authorisation** | RLS is live on every table created. An operator sees exactly what Database Design §7 says they should — verified by test, not by inspection. |
| **Dashboard** | A signed-in operator lands on a shell showing who they are, their roles, and their own families. No metrics, no charts. |
| **Create Family** | A consultant creates a family, becoming its relationship owner. |
| **Add Traveller** | Travellers are added to a family, with family role and nationality. |
| **View Family** | A family detail page showing members, ownership and state. |
| **Audit** | Every write produces a `domain_events` row and correct `created_by` / `updated_by`, without the application being trusted to supply them. |

### What Sprint 1 explicitly does not contain

No trips, enquiries, rates, bookings, documents, tasks, money, intake, review queue, Sabre, conversations or reporting. No search. No file uploads. No dashboards with numbers on them. **Definition of Done §8 is a ceiling, not a floor** — building past it is a failure of the sprint, not a bonus.

### The real deliverable

The patterns. By the end of Sprint 1 these are settled and demonstrated once, correctly:

- how a Server Component reads data through the service layer
- how a Server Action validates with Zod and writes
- how RLS carries authorisation so application code never re-implements it
- how domain events are emitted transactionally
- how errors surface to the user
- how a feature module is laid out

---

# 2. Project Initialization

## 2.1 Next.js

Initialise **Next.js 15 with the App Router**, TypeScript, Tailwind and the `src/` directory, using the ESLint default. React 19 comes with Next 15.

Settings that matter, chosen once:

| Setting | Value | Why |
|---|---|---|
| App Router | Yes | Server Components are the point (Vision technology section) |
| `src/` directory | Yes | Keeps `supabase/`, config and docs clearly outside application code |
| TypeScript strict | Yes, plus `noUncheckedIndexedAccess` | Correctness over convenience |
| Import alias | `@/*` → `src/*` | |
| Turbopack (dev) | Yes | |
| React Compiler | **No** | Not for Sprint 1. Adding a compiler while establishing base patterns confuses cause and effect when something misbehaves. |

## 2.2 Dependencies

Keep the list short. Every dependency is a maintenance obligation for a small team.

**Runtime**
- `@supabase/supabase-js` — the client
- `@supabase/ssr` — cookie-based sessions for App Router. **This, not `auth-helpers`**, which is superseded.
- `zod` — validation, at every boundary
- `react-hook-form` + `@hookform/resolvers` — form state, sharing the Zod schema with the server
- `date-fns` — date handling. No moment, no dayjs.
- `clsx` + `tailwind-merge` — class composition (shadcn dependency)
- `lucide-react` — icons (shadcn default)

**Dev**
- `typescript`, `@types/*`
- `eslint`, `eslint-config-next`
- `prettier` + `prettier-plugin-tailwindcss`
- `supabase` CLI — migrations and type generation
- `vitest` + `@testing-library/react` — unit and component tests
- `@playwright/test` — end-to-end, used in Sprint 1 for the auth and RLS paths

**Explicitly not installed:** any ORM (Prisma, Drizzle), any state manager (Redux, Zustand — Server Components plus URL state cover Sprint 1), any data-fetching library (React Query — Server Components fetch), any component library other than shadcn, Docker, or a local PostgreSQL.

## 2.3 Environment variables

`.env.local`, never committed. `.env.example` **is** committed, with keys and empty values.

| Variable | Exposure | Purpose |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Browser | Hosted project URL |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Browser | Publishable key. Safe to expose **only because RLS is enabled everywhere** — Database Design §7.2. |
| `SUPABASE_SECRET_KEY` | Server only | **Not used in Sprint 1.** Reserved for the intake worker. Bypasses RLS entirely. |
| `NEXT_PUBLIC_SITE_URL` | Browser | Auth redirect target |

Rules:

1. **Never hardcode a key.** Not in a fallback, not in a test, not in a comment.
2. **`SUPABASE_SECRET_KEY` must never gain a `NEXT_PUBLIC_` prefix.** Database Design §7.5 names it the highest-value credential in the system: it bypasses RLS, so exposing it in a browser bundle would defeat every access control in the design.
3. **Validate environment at startup** with a Zod schema in `src/lib/env.ts`, parsed once and exported typed. A missing variable must fail loudly at boot, not produce a confusing runtime error inside a data call.
4. Same variable names in Vercel. Sprint 1 targets a single hosted Supabase project for local and preview alike; a separate staging project is a Sprint 3+ decision.

## 2.4 Tailwind

**Tailwind v4**, CSS-first configuration. Design tokens live in `src/app/globals.css` under `@theme`; there is no `tailwind.config.ts` unless a plugin forces one.

Sprint 1 keeps styling deliberately plain — shadcn defaults, neutral palette, no custom brand work. Vision principle 16 says the interface should be good enough that consultants want to use it, and that it is the ancestor of the client-facing product. **That work is real and it is not Sprint 1**; doing it now would mean redoing it once the product's actual shape appears.

## 2.5 shadcn/ui

Initialise shadcn with the New York style, neutral base colour, CSS variables enabled, and components resolving to `@/components/ui`.

Only the components Sprint 1 actually uses: `button`, `input`, `label`, `form`, `card`, `select`, `dialog`, `sonner` (toasts), `table`, `badge`, `dropdown-menu`, `avatar`, `separator`, `skeleton`.

**shadcn components are ours once installed.** They are copied source, not a dependency. Editing them is expected; wrapping them in a second layer of abstraction is not.

## 2.6 Supabase clients

Three distinct clients, three distinct purposes. Confusing them is the most common App Router mistake, and it fails in ways that look like caching bugs.

| Client | File | Used by | Notes |
|---|---|---|---|
| **Server** | `src/lib/supabase/server.ts` | Server Components, Server Actions, Route Handlers | Reads and writes auth cookies via `next/headers`. Created per request, never cached in a module variable. |
| **Browser** | `src/lib/supabase/client.ts` | Client Components only | Sprint 1 uses this almost nowhere — mutations go through Server Actions. |
| **Middleware** | `src/lib/supabase/middleware.ts` | `middleware.ts` | Refreshes the session cookie on every request and redirects unauthenticated traffic. |

Rules:

1. **Never a module-level singleton for the server client.** It would leak one user's session into another's request.
2. **Nothing outside `src/lib/supabase/` and `src/services/` imports Supabase.** Components never see a Supabase client. This is what makes the service layer the only data boundary.
3. **Types are generated**, never handwritten: `supabase gen types typescript` into `src/types/database.ts`, regenerated after every migration and committed. This is not an ORM — it is the schema telling TypeScript the truth.

---

# 3. Supabase Setup

**The project is an existing hosted Supabase instance. There is no local database, no Docker, no `supabase start`.** The CLI is used only to author migrations, push them to the hosted project, and generate types.

## 3.1 Project connection

- `supabase link` against the existing project ref. It writes `supabase/config.toml`, which **is** committed.
- The database password and access token stay in the developer's environment and in Vercel — never in the repository.
- `supabase db push` applies migrations to the hosted project. `supabase db pull` is the recovery path if someone changes the schema through the dashboard — which should not happen, see §3.4.

## 3.2 Authentication

**Email OTP / magic link only.** No passwords in Sprint 1: fewer failure modes, no reset flow to build, no password storage decisions to defend. Adding a password grant later touches nothing else.

Configuration on the hosted project:

| Setting | Value | Why |
|---|---|---|
| Public signup | **Disabled** | This is an internal platform. Anyone able to sign up would appear as an authenticated principal. |
| Email confirmations | Enabled | |
| Site URL / redirect allow-list | Local, preview and production URLs | |
| Session length | 7 days, refresh rotation on | Consultants work daily; weekly re-auth is not friction |
| JWT expiry | 1 hour | Middleware refreshes it transparently |

**Operator provisioning — the part that matters.** An `auth.users` row does not make someone an operator. The link is deliberate and one-directional:

1. An administrator creates an `identity.operators` row in state `invited`, with the person's work email, before that person exists in `auth.users`.
2. The administrator invites them through Supabase Auth.
3. On first successful sign-in, a trigger matches the authenticated email to an `invited` operator row, sets `auth_user_id`, and moves them to `active`.
4. **If no matching invited operator exists, no operator is created.** The session is valid but the person is nobody, and the application signs them out with a clear message.

This is deny-by-default applied to identity itself. The alternative — auto-creating an operator on first sign-in — would mean a stray auth user silently becomes staff.

The very first operator (the founder or the developer) is created by a **seed script**, run once against the hosted project, matching an auth user made through the dashboard. Documented in the repository, not folklore.

## 3.3 Storage buckets

Sprint 1 uploads no files. Buckets are created anyway, **private from creation**, because a bucket created later under time pressure is a bucket created public.

| Bucket | Access | Used from |
|---|---|---|
| `documents` | Private. Access via short-lived signed URLs only. | Sprint 5 — passports and identity documents |
| `contracts` | Private | Sprint 4 — source rate-contract PDFs |
| `proposals` | Private | Sprint 3 — rendered proposal PDFs |
| `messages` | Private | Sprint 6 — email attachments |

**Storage policies must mirror table RLS** (Database Design §7.5). A document row protected by RLS whose file sits in a readable bucket is not protected. Sprint 1 sets every bucket private with no policies at all, so nothing can be read until the sprint that needs it writes deliberate ones.

## 3.4 SQL migration strategy

**Migrations are the only way the schema changes. No dashboard edits, ever.** A schema change made through the UI exists in exactly one environment and is invisible in review — and this project will have more environments later.

| Rule | Detail |
|---|---|
| Naming | `NNN_description.sql`, zero-padded, in `supabase/migrations/`, matching Database Design §9 phases |
| Immutability | A pushed migration is never edited. Corrections are new migrations. |
| Atomicity | One migration per schema/phase, so a failure leaves a comprehensible state |
| RLS in the same file | Policies ship with the tables they protect — Database Design §9 is explicit: a table that exists without RLS for even one deployment will be queried without it |
| Idempotency | `IF NOT EXISTS` where the semantics allow, so a partial failure can be re-run |
| Types after push | Regenerate `src/types/database.ts` and commit it in the same PR |
| Rollback | **Forward-only.** No down migrations. At this stage a bad migration is fixed by a corrective one; down-migrations create a second, untested path through the schema. |

## 3.5 RLS strategy for Sprint 1

Implementing Database Design §7, in this order — and the order is the point:

1. **`security` schema first, before any business table.** The `STABLE SECURITY DEFINER` helpers — `current_operator_id()`, `has_role()`, `has_sensitive_grant()`, `owns_family()`, `can_see_trip()`. Every policy calls these; none contains inline logic. Database Design §7.2: this is what makes changing the access model, and later adding tenancy or a client role, a change to a handful of functions rather than a hundred policies.
2. **`SECURITY DEFINER` to avoid recursion.** Checking "does this operator hold this role?" must not itself trigger RLS on `identity.operator_roles`.
3. **Enable RLS on every table, including `reference`.** No exceptions — an exception list is a thing people add to.
4. **Default deny.** No permissive catch-all policy anywhere.
5. **Every policy filters `deleted_at IS NULL`** on soft-deletable tables, so soft-deleted rows are invisible without application cooperation.
6. **Sprint 1 introduces no service-role usage at all.** The bypass key stays unused until the intake worker, and its write surface is then bounded by the `intake` schema.

**RLS is tested, not assumed.** Sprint 1 ships a Playwright suite that signs in as two operators with different roles and asserts what each can and cannot read and write. Testing authorisation by reading the policy file is how authorisation bugs reach production.

---

# 4. Migration Plan

Ordered per Database Design §9. **Only migrations 000–004 are applied in Sprint 1.** The rest are listed so the sequence is settled and no later sprint reorders it.

| # | File | Creates | Sprint |
|---|---|---|---|
| 000 | `000_extensions_and_schemas.sql` | Extensions (`pgcrypto`, `pg_trgm`, `unaccent`, `citext`, `pg_cron`); all 16 schemas; **every enum type** from Database Design §1.8; shared trigger functions for `updated_at`, audit capture, immutability enforcement and event emission. Nothing references anything yet — this is the vocabulary the rest of the schema speaks. | **1** |
| 001 | `001_security_functions.sql` | The `security` schema and its helper functions. **Before any table**, because every table's policies call them. | **1** |
| 002 | `002_identity.sql` | T1 `operators`, T2 `operator_roles`, T3 `sensitive_grants`; the first-sign-in linking trigger; RLS. Everything's `created_by` points here. | **1** |
| 003 | `003_events_and_audit.sql` | T50 `domain_events`, T51 `record_changes`, T52 `security_events`; insert-only enforcement. Early, so audit triggers attach as each later table is created rather than leaving a permanent gap at the start of the system's life. | **1** |
| 004 | `004_reference.sql` | T4 `countries`, T5 `destinations`, T6 `destination_document_policies`. Country seed data included; destinations seeded in Sprint 3. | **1** |
| 005 | `005_crm.sql` | T7 `families`, T8 `travellers`, T9 `contact_identities`, T10 `preferences`, T11 `family_links`, T12 `family_merges`. Includes the deferrable `principal_traveller_id` FK and — critically — **the composite unique key `travellers(family_id, id)`** that later migrations depend on to enforce invariant 6. | **1** |
| 006 | `006_rates_supply.sql` | T22–T26: suppliers, contacts, properties, room types, supplier-properties | 4 |
| 007 | `007_rates_contracts.sql` | T27 `rate_contracts`, T28 `rate_lines`; supersession and post-activation immutability triggers | 4 |
| 008 | `008_pipeline.sql` | T14 `loss_reasons`, T13 `enquiries` | 2 |
| 009 | `009_planning_trips.sql` | T15 `trips`, T16 `trip_party`, T17 `trip_destinations`; the composite FK chain making invariant 6 unviolatable | 3 |
| 010 | `010_planning_itinerary.sql` | T18 `itinerary_days`, T19 `itinerary_components`, T20 `component_costs`; the accommodation-overlap exclusion constraint | 3 |
| 011 | `011_planning_proposals.sql` | T21 `proposals`; snapshot immutability; partial unique index enforcing one accepted proposal per trip | 3 |
| 012 | `012_booking.sql` | T29 `bookings`, T30 `booking_travellers` | 4 |
| 013 | `013_sabre.sql` | T31–T34. **Can slip without blocking anything else** — Vision's stated fallback if Sabre access proves harder than assumed. | 5 |
| 014 | `014_documents.sql` | T35 `documents`, T36 `document_access_log`; retention and purge support. Ships with its Storage bucket policies, as one reviewable unit. | 5 |
| 015 | `015_operations.sql` | T37 `task_templates`, T38 `tasks`; template seed data | 5 |
| 016 | `016_money.sql` | T39–T41: schedules, milestones, payments; payment immutability | 5 |
| 017 | `017_intake.sql` | T42–T46: messages, attachments, extractions, auto-accept policies, review queue. Includes the check constraints enforcing domain rules 32 and 33. | 6 |
| 018 | `018_conversations.sql` | T47–T49: threads, communications, notes. Includes the rule 37 constraint — a machine-drafted message cannot reach `sent` without a human approver. | 6 |
| 019 | `019_provenance_links.sql` | Adds `source_review_item_id` to every intake-writable business table. **A deliberate final alteration pass** — Database Design §9 inverts product order here so provenance pointers are not forward references. | 6 |
| 020 | `020_reporting.sql` | Materialized views, their required unique indexes, `pg_cron` refresh schedules | 7 |

**Sprint 1 applies 000 → 005, in two implementation milestones.** Six migrations, one hosted project, pushed in order — but with a hard verification gate in the middle.

### Milestone A — security foundation

| # | File |
|---|---|
| 000 | `000_extensions_and_schemas.sql` |
| 001 | `001_security_functions.sql` |
| 002 | `002_identity.sql` |

**After Milestone A, implementation stops and the following are explicitly verified:**

1. **Hosted Supabase connection works** — the CLI is linked to the existing project, migrations push successfully, and remote migration history matches the repository.
2. **Authentication works** — an invited operator receives a magic link, signs in, and the session persists and refreshes.
3. **Operator provisioning works** — the first-sign-in trigger links an auth user to a pre-existing `invited` operator and activates them; an auth user with no matching invited operator resolves to no operator and creates nothing.
4. **RLS works** — RLS is enabled on every table created so far, `security` helper functions resolve correctly under a real session, and default-deny holds for an unauthenticated request.
5. **Generated database types are correct** — `src/types/database.ts` regenerates from the live schema, compiles, and reflects the enums and tables just created.

**Only after all five checks pass does implementation continue.**

### Milestone B — business tables

| # | File |
|---|---|
| 003 | `003_events_and_audit.sql` |
| 004 | `004_reference.sql` |
| 005 | `005_crm.sql` |

### Why the split

**It reduces debugging complexity.** If authentication, operator resolution, RLS and type generation are all introduced alongside business tables, a failure anywhere in that stack presents identically — an empty result set. Separating them means that when something breaks in Milestone B, the security foundation underneath it is already known to work, and the problem is necessarily in the new tables.

**It validates the security foundation before any business data can exist.** Every RLS policy in this system calls the `security` helper functions, and every table's `created_by` points at `identity.operators`. Introducing CRM tables before those are proven means writing policies against untested primitives — and discovering the primitive was wrong after six tables depend on it, rather than after two.

Note that 005 creates all six CRM tables even though Sprint 1's UI touches only families and travellers. Splitting a schema across sprints creates migrations that depend on each other in ways the phase list does not show. The tables exist, protected by RLS, unused.

---

# 5. Build Order

```
Environment & tooling
        ↓
Database foundation   (extensions, enums, security functions)
        ↓
Identity              (operators, roles, grants)
        ↓
Authentication        (sign-in, session, middleware, sign-out)
        ↓
Application shell     (layout, nav, dashboard)
        ↓
CRM: Family           (create, list, view)
        ↓
CRM: Traveller        (add to family, view)
        ↓
Verification          (RLS tests, audit tests, E2E)
```

Later sprints, fixed now so the sequence is not re-litigated: **Enquiries → Trips & Itinerary → Rates → Bookings → Documents → Operations & Money → Intake & Review Queue → Sabre → Reporting.**

## Why this order

**Foundation before tables.** Enums, security helpers and shared triggers are referenced by everything. Retrofitting an enum onto a populated text column, or adding RLS helpers after fifty policies exist, are both migrations nobody enjoys.

**Identity before authentication.** Authentication needs somewhere to land. Without `identity.operators`, a valid session belongs to nobody and there is no way to express "signed in but not staff" — which is the deny-by-default behaviour §3.2 requires.

**Authentication before anything readable.** Every RLS policy calls `current_operator_id()`. Until sessions work, no policy can be tested, and an untested policy is an assumption.

**CRM before everything else in the product.** Domain Model §11 calls Family the highest-leverage aggregate; Database Design has trips, enquiries, documents and bookings all pointing at families and travellers. Building trips first would mean building against a family model that had never met real data.

**Family before Traveller.** Travellers cannot exist without a family (invariant 1), and the composite unique key created with travellers is what later makes invariant 6 enforceable.

**Verification last within the sprint, not after it.** RLS and audit tests are Sprint 1 deliverables. Deferring them means the pattern every later sprint copies was never proven.

### Why the later order is what it is

Enquiries precede Trips because an enquiry produces a trip and is upstream. **Rates precede Bookings** — Vision sequencing puts rates in Phase 2 as the strategic asset and the module consultants feel first; a booking with no contracted rate to point at is a booking built against a placeholder. Documents and Operations follow trips because both are driven by `TripConfirmed`. **Intake comes late despite being the product's front door**, exactly as Database Design §9 describes: review items name target tables that must exist, and business tables carry provenance pointers back. Sabre can slip. Reporting is last because everything it reads must exist and nothing depends on it.

---

# 6. Repository Structure

```
pureluxe-studio/
├── docs/                                   # The four source documents
│
├── supabase/
│   ├── config.toml                         # Committed. Links to the hosted project.
│   ├── migrations/                         # NNN_description.sql, forward-only
│   └── seed/                               # Reference data + first-operator bootstrap
│
├── src/
│   ├── app/                                # Routes only. Thin.
│   │   ├── (auth)/
│   │   │   ├── sign-in/
│   │   │   └── auth/callback/              # OTP redirect handler
│   │   ├── (app)/                          # Authenticated shell
│   │   │   ├── layout.tsx
│   │   │   ├── dashboard/
│   │   │   └── families/
│   │   │       ├── page.tsx                # List
│   │   │       ├── new/
│   │   │       └── [familyId]/
│   │   │           └── travellers/new/
│   │   ├── layout.tsx
│   │   ├── globals.css                     # Tailwind v4 @theme tokens
│   │   ├── error.tsx  ·  not-found.tsx
│   │
│   ├── features/                           # One folder per bounded context
│   │   ├── identity/
│   │   │   ├── components/
│   │   │   ├── actions/                    # Server Actions
│   │   │   ├── schemas/                    # Zod
│   │   │   └── types.ts
│   │   └── crm/
│   │       ├── components/                 # family-form, traveller-list, ...
│   │       ├── actions/                    # create-family, add-traveller
│   │       ├── schemas/
│   │       └── types.ts
│   │
│   ├── services/                           # The ONLY place Supabase is queried
│   │   ├── identity/
│   │   └── crm/                            # families.ts, travellers.ts
│   │
│   ├── components/
│   │   ├── ui/                             # shadcn. Ours once installed.
│   │   └── shared/                         # App-wide: page-header, empty-state, ...
│   │
│   ├── lib/
│   │   ├── supabase/                       # server.ts · client.ts · middleware.ts
│   │   ├── env.ts                          # Zod-validated environment
│   │   ├── errors.ts                       # Error taxonomy + Result type
│   │   ├── result.ts
│   │   └── utils.ts                        # cn() and genuinely generic helpers
│   │
│   ├── types/
│   │   ├── database.ts                     # GENERATED. Never edited by hand.
│   │   └── domain.ts                       # Domain aliases over generated types
│   │
│   ├── hooks/                              # Client-side only. Nearly empty in Sprint 1.
│   └── middleware.ts                       # Session refresh + route protection
│
├── tests/
│   ├── e2e/                                # Playwright: auth, RLS, family flows
│   └── unit/                               # Vitest: schemas, services, utils
│
├── .env.example                            # Committed, empty values
├── .env.local                              # NEVER committed
└── README.md
```

## Folder responsibilities

| Folder | Owns | Must never |
|---|---|---|
| `app/` | Routing, layouts, page composition, `loading`/`error` boundaries. Pages are thin: read params, call a service, render a feature component. | Contain business logic, query Supabase directly, or hold a form's implementation |
| `features/<context>/` | Everything specific to one bounded context: its components, Server Actions, Zod schemas, context types. **Mirrors the schema and context boundaries from Domain Model §3.** | Import from another feature. Cross-context needs go through `services/` or a shared component |
| `features/*/actions/` | Server Actions: authenticate, validate with Zod, call a service, revalidate, return a typed result | Contain SQL or Supabase calls |
| `services/<context>/` | **The only place Supabase is queried.** One module per aggregate. Returns domain types, never raw rows. Errors are mapped to the taxonomy. | Import React, read cookies directly, format anything for display |
| `components/ui/` | shadcn primitives | Contain business logic or import from `features/` |
| `components/shared/` | App-wide composites with no domain knowledge | Know what a family is |
| `lib/supabase/` | The three clients | Be imported by components |
| `lib/` | Cross-cutting infrastructure: env, errors, Result | Contain domain logic |
| `types/database.ts` | Generated schema types | Be hand-edited — regenerate instead |
| `types/domain.ts` | Readable aliases (`Family`, `Traveller`) over generated row types, so components speak the ubiquitous language rather than `Database['crm']['Tables']['families']['Row']` | Redefine shapes that exist in the schema |
| `hooks/` | Client-only React hooks | Fetch data — Server Components do that |
| `supabase/migrations/` | Forward-only schema history | Be edited after being pushed |

### Why `features/` and `services/` are separate

`services/` is where data access lives and is the *only* importer of Supabase. `features/` is where a context's user-facing behaviour lives. The split means a service can be tested without React, a feature can be tested with a stubbed service, and — the real reason — **the boundary that Domain Model §9 draws between contexts is visible in the import graph.** A CRM component importing a planning service is a lint-visible violation, not a judgement call in review.

---

# 7. Coding Standards

## 7.1 Naming

| Thing | Convention | Example |
|---|---|---|
| Files and folders | `kebab-case` | `family-form.tsx`, `create-family.ts` |
| React components | `PascalCase`, matching the file's purpose | `FamilyForm` in `family-form.tsx` |
| Functions, variables | `camelCase` | `createFamily` |
| Types, interfaces | `PascalCase`. No `I` prefix. | `Family`, `CreateFamilyInput` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_TRAVELLERS_PER_FAMILY` |
| Zod schemas | `camelCase` + `Schema` | `createFamilySchema` |
| Server Actions | Verb-first | `createFamily`, `addTraveller` |
| Service functions | Verb-first, aggregate-scoped | `listFamiliesForOperator`, `getFamilyById` |
| Database | `snake_case` — as designed | `crm.families.relationship_owner_id` |
| Routes | `kebab-case`, plural resources | `/families/[familyId]/travellers/new` |

**The ubiquitous language is enforced here.** `family` not `customer` or `client`. `traveller` not `passenger` or `guest`. `operator` not `user` — `user` is reserved for the future client (Domain Model §2). `booking` not `reservation` — `reservation` means Sabre's record and nothing else. A pull request using a banned term does not merge.

## 7.2 Components

1. **Server Components by default.** `'use client'` requires a reason: state, effects, event handlers, or a browser API. The reason goes in a comment on first use in a file.
2. **Push `'use client'` to the leaves.** A form is client; the page containing it is not.
3. **One component per file**, named for the file.
4. **Props are explicit typed objects.** No prop spreading except into shadcn primitives.
5. **No data fetching in Client Components** in Sprint 1. Server Components fetch; Server Actions mutate.
6. **Composition over configuration.** A component with six booleans wants to be three components.
7. **Every list has an empty state**, every async boundary has a `loading.tsx`, every route segment has an `error.tsx`. Sprint 1 sets this precedent because it is nearly impossible to retrofit consistently.

## 7.3 Service layer

The most important rules in this document, because everything else depends on them holding.

1. **Only `services/` imports a Supabase client.** No exceptions. This is checked by an ESLint import restriction, not by discipline.
2. **One service module per aggregate root**, named for it: `services/crm/families.ts`.
3. **Services return domain types**, never raw query builders and never Supabase's `{ data, error }`.
4. **Services return `Result<T, AppError>`**, not thrown exceptions, for expected failures — not found, forbidden, validation. Unexpected failures throw and reach the error boundary.
5. **Services never re-implement authorisation.** RLS is the authorisation layer (Database Design §7). A service that adds `.eq('relationship_owner_id', operatorId)` "to be safe" is duplicating a rule in a second place where it can drift. **Fail closed instead**: if RLS returns no rows, that is the answer.
6. **Services never compute what the database computes.** No margin, no readiness, no overdue flags in TypeScript (Database Design §5).
7. **Never `select('*')`.** Name columns explicitly, so a schema change surfaces as a type error rather than a silently widened payload — and so sensitive columns are never fetched by accident.
8. **Cross-aggregate reads are separate calls or a database view**, never a service reaching into another context's tables.

## 7.4 Validation

1. **Zod at every boundary**: form input, Server Action arguments, environment variables, and any external payload.
2. **One schema, both sides.** The same Zod schema drives `react-hook-form` and the Server Action. Client validation is a convenience; **the server-side parse is the real one** and runs unconditionally.
3. **Schemas live in `features/<context>/schemas/`** and are named for the operation: `createFamilySchema`.
4. **Schemas mirror database constraints.** If the column is `NOT NULL` with a check, the schema says so. Divergence produces errors from PostgreSQL rather than from the form, which users experience as the application breaking.
5. **Infer types from schemas** — `z.infer` — rather than declaring a type and a schema that can disagree.
6. **Never trust a client-supplied identity.** `created_by`, `relationship_owner_id` and `updated_by` come from the session on the server, never from the form payload.

## 7.5 Error handling

A small explicit taxonomy in `lib/errors.ts`:

| Error | Meaning | Surfaces as |
|---|---|---|
| `ValidationError` | Input failed Zod | Inline field messages |
| `NotFoundError` | Row absent, or invisible under RLS | 404 page. **Deliberately indistinguishable** — "forbidden" leaks existence |
| `ForbiddenError` | Authenticated, authorised action refused | Explanatory message |
| `UnauthenticatedError` | No valid session | Redirect to sign-in |
| `ConflictError` | Unique or state-transition violation | Actionable message ("a family with that identity already exists") |
| `DatabaseError` | Unexpected PostgreSQL failure | Generic message, full detail logged |

Rules:

1. **Expected failures are `Result` values; unexpected failures throw.** A family that does not exist is expected. A dropped connection is not.
2. **Never render a raw PostgreSQL error.** Constraint names are internal detail; some carry column values.
3. **Map database error codes to the taxonomy in the service layer** — `23505` → `ConflictError`, `23503` → `ValidationError`, `42501` → `NotFoundError`. Callers never see a PostgreSQL code.
4. **Log with correlation.** Every server-side error logs the operator id and a correlation id — the same one written to `domain_events.correlation_id`, so a user report can be traced to what actually happened.
5. **Toasts confirm, error boundaries recover.** A failed mutation shows a toast and keeps the form's state; a failed page render hits `error.tsx` with a retry.

## 7.6 TypeScript

1. **`strict: true`, plus `noUncheckedIndexedAccess`.**
2. **`any` is banned.** `unknown` plus narrowing where a type is genuinely unknown.
3. **No non-null assertions (`!`)** outside generated code. If it cannot be null, the type should say so; if it can, handle it.
4. **Type assertions require a comment** explaining why the compiler cannot see what you can.
5. **Types derive from the schema.** `types/domain.ts` aliases generated row types; it does not restate them.
6. **Discriminated unions over optional-field soup** — especially `Result`.
7. **`satisfies` over `as`** for config objects.
8. **Enums come from the database.** The generated types expose PostgreSQL enums; do not declare a parallel TypeScript enum that can drift from `trip_state`.

## 7.7 Writes, events and audit

Implementing Database Design §6.3 — every meaningful state change writes a `domain_events` row **in the same transaction**. `supabase-js` cannot wrap two statements in one transaction from the application, so:

| Write shape | Mechanism | Why |
|---|---|---|
| **Simple create / update / delete** (Sprint 1: family, traveller) | **Database trigger** emits the event | Transactional by construction. The application cannot forget, and cannot lie about the actor. |
| **State transitions where intent matters** (later: `TripConfirmed` vs a generic update) | **A PostgreSQL function called via RPC** | The event *type* carries business intent a trigger cannot infer from row diffs |
| **Multi-aggregate operations** (later: confirming a trip, applying a review item) | RPC function | Atomicity across tables |

Sprint 1 needs only the trigger path, but the RPC pattern is documented now so Sprint 3 does not invent a different one.

`created_by` and `updated_by` are set **by trigger** from `security.current_operator_id()`, never from application input. An actor the client can supply is an actor the client can forge.

## 7.8 Git and review

- Branch per feature: `feat/crm-create-family`. Never commit to `main`.
- Conventional commits: `feat(crm):`, `fix(auth):`, `chore(db):`.
- **A migration and its regenerated `database.ts` land in the same pull request.** They are one change.
- Review checklist: ubiquitous language correct · no Supabase import outside `services/` · Zod on every boundary · RLS policy present for every new table · nothing derived stored · no secret in the diff.

---

# 8. Definition of Done

Sprint 1 is complete when all of the following are true on the hosted Supabase project, verified by running them — not by reading the code.

### Functional

- [ ] **Login** — an invited operator receives a magic link, signs in, and lands on the dashboard. Session survives a refresh and a restart.
- [ ] **Rejected login** — a valid auth user with no matching invited operator is signed out with a clear message and creates nothing.
- [ ] **Dashboard** — shows the operator's name, roles, and their own families. No metrics.
- [ ] **Create Family** — a consultant creates a family and becomes its `relationship_owner_id`. Validation errors render inline.
- [ ] **Add Traveller** — a traveller is added to a family with role and nationality. Invariant 1 holds structurally.
- [ ] **View Family** — a detail page shows the family, its state, its owner, and its travellers, with a working empty state.
- [ ] **Sign out** — clears the session and blocks protected routes.

### Technical

- [ ] Migrations 000–005 applied to the hosted project, in order, forward-only.
- [ ] RLS enabled on **every** table created, with default-deny policies.
- [ ] `security` helper functions in place; **no policy contains inline logic**.
- [ ] `src/types/database.ts` generated from the live schema and committed.
- [ ] Every write produces a `domain_events` row with the correct actor, event type and correlation id.
- [ ] `created_by` / `updated_by` populated by trigger, never by application input.
- [ ] Zod validation on every Server Action, parsed server-side unconditionally.
- [ ] No Supabase import outside `src/lib/supabase/` and `src/services/`, enforced by lint.
- [ ] No secret in the repository. `.env.example` committed with empty values.
- [ ] Deployed to Vercel with environment variables set; the deployed app performs the full flow.

### Verified by test

- [ ] **E2E (Playwright):** sign in → dashboard → create family → add traveller → view family → sign out.
- [ ] **RLS (Playwright, two operators):** consultant A cannot write to consultant B's family; both can read it (deliberate cross-visibility per Database Design §7.3); a signed-out request reads nothing.
- [ ] **Audit:** creating a family produces exactly one `FamilyCreated` event with the right actor.
- [ ] **Unit (Vitest):** Zod schemas accept valid and reject invalid input at the boundaries the database also enforces.

### Explicitly not done — and that is correct

No trips, enquiries, rates, bookings, documents, tasks, money, intake, Sabre, conversations, reporting, search, file upload, or dashboard metrics. **If any of these exist at the end of Sprint 1, the sprint failed**: the scope was a ceiling, and the patterns being proven are worth more than the features not yet built.

---

# 9. Sprint Checklist

Chronological. Each item is verifiable. Nothing is started before the item above it is done.

### Phase A — Project setup
1. [ ] Initialise Next.js 15 (App Router, TypeScript, Tailwind, `src/`, `@/*` alias, no React Compiler)
2. [ ] Configure `tsconfig.json`: `strict`, `noUncheckedIndexedAccess`
3. [ ] Install runtime dependencies (§2.2)
4. [ ] Install dev dependencies; configure ESLint, Prettier with the Tailwind plugin
5. [ ] **Add the ESLint import restriction** banning Supabase imports outside `lib/supabase/` and `services/` — before there is code to violate it
6. [ ] Initialise shadcn/ui; add only the Sprint 1 component list (§2.5)
7. [ ] Create `.env.example`; create local `.env.local` with the hosted project's URL and publishable key
8. [ ] Build `src/lib/env.ts` — Zod-validated, parsed once, fails loudly at boot
9. [ ] Create the folder structure from §6, with `.gitkeep` where empty
10. [ ] `git init` equivalent housekeeping: `.gitignore` covers `.env.local`; first commit

### Phase B — Supabase connection
11. [ ] `supabase link` to the existing hosted project; commit `config.toml`
12. [ ] Confirm CLI connectivity by listing remote migrations. **No `supabase start`, no Docker, no local database.**
13. [ ] Configure Auth on the hosted project: magic link on, **public signup off**, redirect URLs, session length (§3.2)
14. [ ] Create the four Storage buckets, all private, no policies (§3.3)
15. [ ] Write the three Supabase clients: `server.ts`, `client.ts`, `middleware.ts` (§2.6)

### Phase C — Database foundation, Milestone A *(security foundation)*
16. [ ] Write and push `000_extensions_and_schemas.sql` — extensions, 16 schemas, every enum, shared trigger functions
17. [ ] Write and push `001_security_functions.sql` — the `security` schema helpers
18. [ ] Write and push `002_identity.sql` — operators, roles, grants, first-sign-in linking trigger, RLS
18a. [ ] Generate `src/types/database.ts` from the live schema; confirm it compiles
18b. [ ] Seed the first operator: create the auth user in the dashboard, run the bootstrap seed, verify the linking trigger activates them on first sign-in

### GATE A — verify before continuing
**Implementation stops here. All five checks must pass before any Milestone B work begins.**

- A.1 [ ] **Hosted Supabase connection works** — CLI linked to the existing project, migrations pushed, remote migration history matches the repository
- A.2 [ ] **Authentication works** — an invited operator receives a magic link, signs in, session persists and refreshes
- A.3 [ ] **Operator provisioning works** — first-sign-in trigger links and activates an invited operator; an auth user with no matching invited operator resolves to no operator and creates nothing
- A.4 [ ] **RLS works** — enabled on every table created so far, `security` helpers resolve correctly under a real session, default-deny holds for an unauthenticated request
- A.5 [ ] **Generated database types are correct** — `src/types/database.ts` reflects the enums and tables just created, and compiles

*Note: Phase D (Authentication) is required to satisfy A.2–A.4. Execute Phase D before this gate, then return here.*

### Phase C — Database foundation, Milestone B *(business tables)*
19. [ ] Write and push `003_events_and_audit.sql` — event log, change history, security events, insert-only enforcement
20. [ ] Write and push `004_reference.sql` — countries (seeded), destinations, destination policies
21. [ ] Write and push `005_crm.sql` — all six CRM tables, the deferrable principal-traveller FK, **the composite unique key on `travellers(family_id, id)`**, RLS
22. [ ] Regenerate `src/types/database.ts`; commit it with the migrations
23. [ ] Verify RLS and default-deny on all six CRM tables under a real session

### Phase D — Authentication
24. [ ] `middleware.ts`: refresh session, protect the `(app)` route group, redirect unauthenticated traffic to sign-in
25. [ ] Sign-in page: email input, Zod-validated, magic-link request, clear pending and error states
26. [ ] Auth callback route: exchange the code, establish the session, redirect
27. [ ] **Operator resolution**: after sign-in, resolve the session to an operator. No operator → sign out with a clear message and create nothing (§3.2)
28. [ ] `services/identity/operators.ts`: `getCurrentOperator()` returning operator plus roles
29. [ ] Sign-out action
30. [ ] Manually verify: sign in, refresh, restart the server, sign out, attempt a protected route while signed out

### Phase E — Application shell
31. [ ] `(app)/layout.tsx`: authenticated shell, nav, current operator in the header
32. [ ] Shared components: `page-header`, `empty-state`, `loading-skeleton`
33. [ ] `error.tsx` and `not-found.tsx` at the root and in the `(app)` group
34. [ ] Dashboard page: operator name, roles, own families. **No metrics** — Vision principle that a visibly wrong number destroys trust permanently, and there is nothing worth counting yet
35. [ ] Toast provider wired for action feedback

### Phase F — CRM: Family
36. [ ] `lib/result.ts` and `lib/errors.ts`: `Result` type and the error taxonomy (§7.5)
37. [ ] `types/domain.ts`: `Family`, `Traveller` aliases over generated row types
38. [ ] `services/crm/families.ts`: `listFamiliesForOperator`, `getFamilyById`, `createFamily` — returning `Result`, mapping PostgreSQL error codes, **no `select('*')`**
39. [ ] `features/crm/schemas/`: `createFamilySchema`, mirroring the database constraints
40. [ ] `features/crm/actions/create-family.ts`: authenticate → Zod parse → service → revalidate → typed result
41. [ ] `features/crm/components/family-form.tsx`: client component, `react-hook-form` with the shared schema
42. [ ] `/families` list page with an empty state
43. [ ] `/families/new` page
44. [ ] `/families/[familyId]` detail page, 404 for a family RLS does not expose
45. [ ] Verify `created_by`, `relationship_owner_id` and the `FamilyCreated` event are all correct after a create

### Phase G — CRM: Traveller
46. [ ] `services/crm/travellers.ts`: `listTravellersForFamily`, `addTraveller`
47. [ ] `features/crm/schemas/`: `addTravellerSchema` — family role, nationality, minor rule mirroring invariant 4
48. [ ] `features/crm/actions/add-traveller.ts`
49. [ ] `features/crm/components/traveller-form.tsx` and `traveller-list.tsx`
50. [ ] `/families/[familyId]/travellers/new` page
51. [ ] Traveller list on the family detail page, with an empty state
52. [ ] Verify invariant 1 holds and the `TravellerAdded` event is emitted

### Phase H — Verification
53. [ ] Vitest: Zod schema unit tests, valid and invalid, at boundaries the database also enforces
54. [ ] Vitest: service tests against a stubbed Supabase client
55. [ ] Playwright: full happy path — sign in → create family → add traveller → view → sign out
56. [ ] **Playwright: RLS tests with two operators** — A cannot write to B's family; both can read it; signed-out reads nothing
57. [ ] Playwright: audit assertions — one `FamilyCreated` event, correct actor
58. [ ] Manual pass over the §8 checklist, item by item, on the deployed application

### Phase I — Deploy
59. [ ] Connect the repository to Vercel; set all environment variables (no secret key yet)
60. [ ] Deploy to preview; run the full flow against it
61. [ ] Confirm the redirect allow-list includes the preview and production URLs
62. [ ] Deploy to production; run the flow once more
63. [ ] README: setup, environment, migration workflow, first-operator bootstrap
64. [ ] Tag the release and open Sprint 2

---

## Appendix — Decisions this plan makes that the source documents did not

Flagged for review, since each is a delivery choice rather than an implementation of an approved decision.

1. **Magic-link auth, no passwords.** Fewer failure modes and no reset flow. Adding a password grant later is isolated.
2. **Operators are invited, never self-created.** A trigger links an auth user to a pre-existing `invited` operator; no match means no operator. Deny-by-default applied to identity itself.
3. **Domain events emitted by trigger for simple writes, by RPC for state transitions.** `supabase-js` cannot express a multi-statement transaction, and Database Design §6.3 requires the event and the change to be atomic.
4. **Forward-only migrations, no down-migrations.** A down-migration is a second, untested path through the schema.
5. **All six CRM tables ship in Sprint 1** even though only two have UI. Splitting a schema across sprints creates dependencies the phase list does not show.
6. **Buckets created private in Sprint 1, with no policies**, years before they are used — because a bucket created later under pressure is a bucket created public.
7. **Services return `Result` for expected failures and throw for unexpected ones**, rather than throwing for everything.
8. **`NotFoundError` and `ForbiddenError` are indistinguishable to the client.** Distinguishing them leaks the existence of records RLS is hiding.
9. **One hosted Supabase project for local, preview and production in Sprint 1.** A separate staging project is worth doing before real client data exists — recommend Sprint 3, and worth an explicit decision then rather than a drift into production-as-development.

---

# 10. Implementation Workflow

**This section is the mandatory execution contract for Claude Code. It overrides any default working style.**

## The rules

1. **Implement exactly ONE checklist item at a time.** One item from §9, start to finish, and nothing else.

2. **Never combine multiple checklist items into a single implementation.** Two items that look adjacent, trivial, or naturally paired are still two items. Convenience is not a reason to merge them.

3. **After completing a checklist item, stop and report:**
   - **What changed** — files created or modified, migrations pushed, configuration altered.
   - **Why it was implemented that way** — the reasoning, and which source document or plan section it follows.
   - **How to verify it locally** — concrete steps the user can run to confirm it works.
   - **Any required commands** — exact commands to run, including anything that must be done in the Supabase dashboard or Vercel.
   - Then **wait for user approval before continuing.**

4. **Never continue automatically.** Do not begin the next checklist item until the user explicitly approves. Silence is not approval. A completed item is a full stop, not a pause.

5. **Never skip checklist items.** The order in §9 is a dependency order, not a suggestion. If an item appears unnecessary, say so and wait — do not skip it unilaterally.

6. **The checklist is the implementation contract.** §9 defines what gets built and in what sequence. Work not on the checklist does not get done in Sprint 1. If something genuinely missing is discovered, raise it and wait for the user to decide whether it joins the checklist.

## Two additional conditions specific to this sprint

- **GATE A is a hard stop.** All five checks must pass before any Milestone B item begins. A gate check that fails is reported and resolved; it is never noted and passed over.
- **Definition of Done (§8) is a ceiling.** Building beyond it is a failure of the sprint, not a bonus. If a checklist item seems to invite adjacent functionality, it does not.
