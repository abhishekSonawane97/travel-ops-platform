# PureLuxe Studio — Product Vision

**For:** CTO review, pre-build
**Date:** 25 July 2026
**Revision:** v2 — adds Email Extraction Pipeline, AI Review Queue, Family Management, Sabre Sync layer; promotes Offline Rates to a strategic asset.

---

## 1. Executive Summary

PureLuxe Studio is the internal operating system for a luxury travel agency.

Today the agency runs on Gmail, WhatsApp, Excel, PDFs and sticky notes. Each works. Together they mean **no system knows the truth about a family, a trip, or a booking** — that lives in whichever consultant's head and phone is handling the file. Fine at today's size. Not survivable past it.

The platform gives every family, traveller, enquiry, trip, booking, rate, document and task one home. Its users are travel consultants, ops, admins and the founder. No clients, not yet.

Three ideas carry the product:

**1. Work starts in the inbox, so the product starts there too.** Rate sheets, supplier confirmations, PNRs, enquiries, passport scans — nearly everything the agency needs to know arrives as email. The **Email Extraction Pipeline** reads that mail and proposes structured records. It is the front door, not a side feature.

**2. Nothing the AI proposes is trusted until a human accepts it.** Every extraction lands in the **AI Review Queue**, where an operator confirms, corrects or rejects it in seconds. This is what makes an AI-first intake safe in a business where a wrong rate or a missed passport expiry costs a client relationship. It is also the honest answer to "when is the AI good enough?" — the queue measures it every day.

**3. The agency's two real assets are its family memory and its offline rates.** Knowing that a client's wife dislikes overwater villas, and holding a contracted Udaipur rate 18% below public, is what an OTA cannot copy. Both currently live in inboxes and stale spreadsheets. Making them structured and searchable *is* the product.

Those three also happen to be the preconditions for the founder's twelve-month goal: a WhatsApp-first, AI-assisted client experience. An AI is only as good as the data it stands on, and only as trusted as its error rate is known. Building internal-first is the path to the OTA, not a detour from it.

**In one line:** *Every trip is planned, sold, operated and remembered in one system — so that a year from now, an AI can help do the same.*

---

## 2. Product Goals

**G1 — Email stops being a filing system.** Inbound mail is read by the extraction pipeline and turned into proposed records. Consultants review, not re-type.

**G2 — Every AI output passes a human.** One review queue for all extractions, cleared daily, with visible accuracy per extraction type.

**G3 — One record per family.** Families, travellers, preferences, history, documents and communications in one place. Nobody re-asks a client what the agency already knows.

**G4 — Offline rates become searchable in seconds.** Contracted rates queryable by destination, property, dates and room category — replacing "which Excel had the Amanbagh rates?"

**G5 — Every enquiry is tracked from first contact to post-travel.** No enquiry lives only in an inbox. The founder sees pipeline without asking anyone.

**G6 — Itineraries are data, not documents.** Client PDFs are rendered output. Change the trip, the proposal changes.

**G7 — Nothing is forgotten.** Confirmed trips generate their own deadlines — visa, passport expiry, deposit, final payment, supplier confirmation. Overdue work surfaces on its own.

**G8 — Passports are handled properly.** One vault, access-controlled, expiry-aware, audited. Never a WhatsApp thread.

**G9 — Sabre stays in sync without anyone re-keying it.** A real sync layer with reconciliation state, not a copy-paste habit.

**G10 — The founder can see the business while it runs.** Pipeline, conversion, margin, trips at risk — readable, not assembled monthly.

Underneath all ten: **capture structure at the moment of entry** (anything the AI will later reason over), and **record who did what** (trust today, grounding later).

---

## 3. Non Goals

These are decisions, not backlog.

- **No client-facing surface.** No client login, no public search, no self-service booking.
- **Not a Sabre replacement.** Sabre stays authoritative for GDS air. We sync and reconcile.
- **No live inventory or booking engine.** No rate shopping, no automated supplier booking. Confirmation stays human.
- **Not accounting.** Trip-level cost/sell/margin/payments only. Invoices, ledgers and GST stay where they are.
- **No marketing automation.** No campaigns, no lead scoring.
- **AI never commits.** *(revised from v1 "no AI in the critical path")* AI proposes; a human accepts. Every workflow must remain completable by hand if extraction fails or is wrong.
- **Single workspace.** One agency. Don't pay the multi-tenancy tax; don't make it impossible later.
- **Responsive web only.** No native app.
- **No workflow builder.** The agency's process is opinionated and encoded. No custom fields, custom roles or configurable pipelines.

---

## 4. Business Problems

**The inbox is the real system of record — and it can't be queried.** Rate sheets, confirmations, PNRs, passports and enquiries all arrive by email and stay there. Everything downstream is someone manually copying out of Gmail. This is the root cause under most of the problems below, which is why the extraction pipeline sits at the front of the product.

**Family knowledge is fragmented and personally owned.** A family's history spans one consultant's Gmail, a WhatsApp thread, an Excel row and their memory. Repeat clients get re-interviewed on things the agency already knows — in luxury travel that's a service failure, not a nuisance. A consultant leaving takes relationships with them.

**Families aren't modelled at all.** The agency sells to households and multi-generational groups, but tracks individuals. The same three passports get re-collected, a child's preferences sit under a parent's name, and "the Mehtas" as an account with a lifetime value exists only in someone's head.

**Bookings are tracked manually.** Per-consultant spreadsheets in divergent formats. No reliable answer to "what's confirmed for August?", no early warning on unpaid deposits, and month-end is a reconstruction.

**Passports are scattered and ungoverned.** WhatsApp images and mail attachments in personal drives and phone galleries. Hard to find under pressure, expiry caught late or by the client, and real unmanaged data-protection liability.

**Deadlines depend on memory.** Deposits, visa lead times, supplier follow-ups. Misses cost held rates, rushed visas and occasionally a relationship worth far more than the trip.

**Supplier conversations are invisible.** Individual email and WhatsApp threads. No shared record of what was asked, held or confirmed; no supplier history; no clean handover.

**Offline rates — the biggest asset, least usable.** Negotiated and contracted rates are the agency's actual edge over public OTAs, and they sit in PDFs and Excel files of varying vintage. Consultants quote from memory or the most recent file they can find. New consultants can't access this knowledge at all, and nobody can tell which contracts earn their keep.

**Sabre drifts.** Bookings made in Sabre are re-typed into trackers. Duplicated effort, transcription errors, two systems silently disagreeing.

**The founder flies blind.** Pipeline, load, conversion, margin and at-risk trips are assembled by asking people. Management by anecdote, weeks behind reality.

---

## 5. Success Metrics

Baselines captured in the first two weeks; targets at six months.

**Pipeline and queue** — the new instrumentation, and the leading signal for the whole AI strategy:
- Inbound mail auto-classified into a proposed record: **≥ 80%**
- Extraction acceptance rate (accepted without correction), by type: **≥ 85% for rates and PNRs, ≥ 90% for passports**
- Review queue cleared daily: **≥ 95% of items reviewed within 24h**
- Queue backlog trend: flat or falling
- Time to review one item: **< 20 seconds median**

**Adoption** — if these fail, nothing else matters:
- Enquiries created in-platform within 24h: **≥ 95%**
- Confirmed bookings with no platform record: **< 2%**
- Weekly active consultants: **100%**
- Trips still tracked in personal Excel: **0 by month 4**

**Speed:**
- Enquiry → first proposal sent: **↓ 40%**
- Time to find a contracted rate: **< 30 seconds** (from minutes)
- Admin and document-chasing time per trip: **↓ 30%**

**Reliability:**
- Overdue tasks at any moment: **< 5%** of open tasks
- Trips departing with all traveller documents on file 14+ days out: **≥ 95%**
- Passport-expiry surprises inside 30 days of departure: **0**
- Sabre records diverging from platform state, unreconciled > 24h: **< 1%**

**Commercial:**
- Enquiry → booking conversion: **↑ 10%** relative
- Confirmed trips with cost, sell and margin recorded: **≥ 98%**
- Monthly reporting: hours → minutes

**Readiness for the client-facing phase** — no operational value today; report monthly anyway, because these decide whether year two is real:
- Families with structured preference profiles: **≥ 80%** of active families
- Frequently sold properties with structured, in-date rates: **≥ 90%**
- Client communications captured in-platform: **≥ 70%**
- Itineraries generated from structured trip data: **100%**

---

## 6. User Personas

**Priya — Travel Consultant.** *Primary persona.* Owns families end to end: enquiry, design, quoting, selling, and being the person they call. Runs 15–25 live files, laptop for work, phone for WhatsApp at all hours, judged on revenue and happy clients. She wants to answer fast, look brilliant, and never be caught not knowing something about her family. She hates rebuilding the same itinerary, hunting last year's rate sheet, and re-typing traveller details into three places. **She'll abandon the platform the first time it's slower than the WhatsApp thread it replaced** — so every core action needs to be a few clicks with good defaults. Success: she opens the Mehta family and knows everything, including what a colleague did two years ago, in fifteen seconds.

**Rahul — Operations Executive.** Turns sold trips into delivered trips: supplier confirmations, documents, payment milestones, final packs, in-trip problems. 40+ trips at different stages, deadline-driven, absorbs the consequences of anything missed. He wants one screen showing what's due and what's at risk. He hates WhatsApp forwards with no context, and discovering on visa-filing day that a passport expires in four months. **Rahul is the review queue's main operator and the persona most likely to love this product** — treat him as the internal champion.

**Anjali — Founder.** Owns commercial performance, key supplier and client relationships, and the push toward the AI-assisted OTA. Mostly out of the office, reads on a phone in short windows, doesn't want to operate the system daily — wants to see through it. She wants pipeline, margin and at-risk trips without interrupting anyone. **One visibly wrong dashboard number and she never trusts it again**, so accuracy beats breadth. Success: she answers "how's August looking?" from her phone, correctly, in a minute.

**Sameer — Administrator.** Users, roles, permissions, and the reference data the platform runs on — suppliers, properties, rate contracts, templates. Probably a senior ops person wearing a second hat, not a dedicated hire. Low tolerance for fiddly config. He wants to onboard people cleanly, keep master data trustworthy, and see what's gone stale without asking engineering. The extraction pipeline is largely *for* him: it turns rate-contract loading from transcription into review.

---

## 7. User Roles

Four roles. Access comes from three separate things — **role** (what work you do), **assignment** (is this family yours), and **sensitivity** (passports and margin are gated regardless of role). Conflating them is the expensive mistake.

**Travel Consultant** — full access to families and trips they own or are assigned; read access to colleagues' trips, because this business loses far more to siloing than to snooping. Creates enquiries, trips, quotes, bookings, communications. Reads rates, can't edit contracts. Sees margin on own trips. Reviews queue items relating to their families.

**Operations Executive** — access across all trips in delivery, regardless of consultant. Owns tasks, supplier confirmations, documents, payment milestones, travel packs. Full document-vault access (it's the job — the control is the audit trail, not a lock). Primary review-queue operator. Limited margin visibility. Can't change commercial terms or ownership.

**Founder** — reads everything including commercials and analytics. Can reassign family and trip ownership. Not expected to do daily data entry.

**Administrator** — user lifecycle, roles, permissions, master data, extraction rules, audit logs. **Not** automatically granted passport access — admin power and data access stay separate.

Three rules for the model: roles are additive, not exclusive (one person can be Founder and Admin — model capability, not job title); sensitive tiers are deny-by-default and every access is logged; and the model must be able to accept **a fifth actor that doesn't exist yet — the client** — without restructuring. That last one is the biggest role-related risk to the year-two plan. No custom roles in v1.

---

## 8. Current Workflow

An enquiry today:

**Arrives** by email, WhatsApp to a personal number, Instagram DM or a referral call — and lands wherever it lands. No shared record it exists; whether it's worked depends on someone noticing.

**Qualification** happens on WhatsApp. Dates, party, budget, preferences — now living only in that thread on that phone. If they're repeat clients, the consultant tries to remember, or searches Gmail.

**Research** means checking Sabre for air, digging through mail attachments and drives for contracted rates, and messaging hotel and DMC contacts for availability and holds. Replies trickle back across threads over one to three days. Some rates are quoted from memory.

**Proposal** gets built in Word or PowerPoint, usually by copying a previous client's document. Pricing in Excel. Exported to PDF, sent by mail or WhatsApp. Changes produce v2, v3, v4 as separate files, and which one the client is actually looking at is guesswork.

**Confirmation** is verbal or a message. Supplier confirmed, air booked in Sabre, details transcribed into a personal tracker, deposit requested by email.

**Handover to ops** is a WhatsApp message or a chat at the desk. Everything never written down is lost here. **Most failures start at this boundary.**

**Documents** are chased over WhatsApp, arrive as images at random hours, get saved to a personal drive or left in the chat. Expiry checking is manual.

**Pre-departure**, ops assembles vouchers and the final itinerary from several sources and re-verifies against supplier email. Payment milestones live in reminders.

**In-trip and after**, issues go to whoever the client messages. Feedback, preferences learned during travel, and supplier performance are almost never written down anywhere durable.

Three structural problems: information is silently lost at **intake** and at **handover**; nothing compounds, so the tenth Maldives itinerary costs nearly what the first did; and the business can only be reconstructed afterwards, never observed while running.

---

## 9. Proposed Workflow

Same journey, one place. The shape is deliberately preserved — consultants aren't asked to work differently, just to work in one system.

**0 — The inbox feeds the platform.** Connected mailboxes are read continuously. The pipeline classifies each message — enquiry, rate sheet, supplier confirmation, PNR, passport, payment advice — extracts the fields that matter, matches it to an existing family, trip or supplier where it can, and files a proposal in the review queue. Attachments are parsed, not just stored. *Nothing is created from mail without a human accepting it.*

**1 — Review, don't re-type.** An operator opens the queue and works items in seconds: accept, correct, or reject. High-confidence, low-risk items (a supplier confirmation matching an existing hold) can be pre-accepted with a visible audit trail; anything touching money, identity or rates always needs a person. **Corrections are the feedback loop** — they're what tells us extraction quality is improving, per type, with real numbers.

**2 — Enquiry captured.** Whatever the channel, an enquiry record exists at first contact, with source, family (matched or new), destinations, dates, party and budget, and an explicit owner. Mail-borne enquiries arrive here via the queue; WhatsApp and phone enquiries are entered by hand. No enquiry exists only in an inbox.

**3 — Family context.** Opening the family shows members and travellers, preferences at family and person level, past trips and spend, documents on file, and the full communication history — including work other consultants did. Repeat clients are never re-interviewed.

**4 — Trip design.** The consultant composes a trip from structured components — accommodation, flights, transfers, experiences, day segments — pulling from past trips and a reusable library. Contracted rates are searched in-platform by destination, property, dates and room category, with contract validity and margin shown at the point of choosing, not checked afterwards.

**5 — Supplier engagement.** Requests for availability, holds and quotes are raised against the trip and supplier. Replies come back through the mail pipeline, get matched to the request in the queue, and land on the thread. Any colleague can see what was asked, of whom, when, and what came back.

**6 — Proposal.** Rendered from trip data, never authored separately. Versions tracked; what was sent and when is unambiguous. **This is the most important architectural commitment in the product.**

**7 — Confirmation.** The trip moves to Confirmed with cost, sell, margin and a payment schedule. Air booked in Sabre appears through the sync layer rather than being re-keyed.

**8 — The operational plan writes itself.** Confirmation generates the task set — supplier confirmations, document collection, visa lead times, deposit and final payment — with dates derived from travel dates and destination rules. Nobody has to remember to make a checklist.

**9 — Handover is a state change, not a conversation.** Ops picks the trip off a work queue with the context already there.

**10 — Documents.** Requested, uploaded (or extracted from mail and confirmed in the queue), stored in the vault against the **traveller** — travellers recur, trips don't — with expiry tracked and checked against travel dates automatically. Passport risk is an alert, never a discovery.

**11 — Pre-departure.** The travel pack generates from the same trip data that produced the proposal and the bookings. Consistency is structural, not proofread.

**12 — After travel.** Issues logged against the trip. Feedback, newly learned preferences and supplier performance land back on the family, traveller and supplier records — so the eleventh Maldives trip is genuinely cheaper to build than the tenth.

**What actually changes:** work compounds. Every trip enriches families, rates, components and supplier history. The system gets more valuable per trip, not just bigger.

---

## 10. Product Principles

Decision rules, in priority order.

**1. Adoption beats completeness.** A feature nobody uses is negative value. Ship the narrow thing consultants use daily before the complete thing they avoid.

**2. Faster than the thing it replaces.** Every core action must beat WhatsApp, Gmail or Excel on time-to-done. Consultants only try once.

**3. The inbox is the front door.** Assume information arrives as mail. Design intake around parsing and proposing, not around forms — but never make a form the only way to fail.

**4. AI proposes, humans dispose.** Nothing an AI extracts or drafts becomes real without a person accepting it. Every workflow stays completable by hand. This is not a temporary safety measure; it's the operating model.

**5. AI raises the consultant's ceiling, it doesn't lower their headcount.** Automate transcription, chasing, searching and checking — the work nobody wants. Leave taste, relationships and judgement alone. In luxury travel the human relationship *is* the product; a consultant freed from re-typing sells more trips, not fewer.

**6. Structure at capture, not extraction later.** Anything we'll query, report on or reason over gets captured structured. Retro-fitting structure onto free text is the cost everyone underestimates, and it's exactly the tax that would make the AI phase unaffordable.

**7. Families are the unit, people are the members.** Model the household, not just the individual. Preferences, documents, history and value belong at the level that matches how the agency actually sells.

**8. Rates are an asset, not a lookup table.** Treat the contract repository as inventory the agency owns. Coverage, freshness and searchability are product metrics, not data-entry chores.

**9. One record, one home.** Every real entity has one authoritative record; everything else points at it. Duplicate entry is a bug.

**10. Documents are outputs, never sources.** Proposals, vouchers and packs render from trip data. No workflow depends on something that only exists inside a generated PDF.

**11. Nothing depends on memory.** If a deadline matters, the system owns it.

**12. Sensitive data is governed from day one.** Passports and personal data get access control, audit, expiry-awareness and retention limits in v1. Legal obligation and trust obligation, not v2 hardening.

**13. Sync with external systems, don't reimplement them.** Sabre and suppliers stay authoritative for what they own. We hold a reconciled view and make disagreement visible. Resist becoming a GDS.

**14. Everything meaningful is attributable.** Who did what, when — including which AI proposed it and which human accepted it.

**15. Build so the client can be added later, not so they're added now.** Keep identity, permissions, communications and trip state clean enough that a client could one day read and write here. Build none of it yet.

**16. Boring technology, beautiful product.** Next.js, TypeScript, Supabase/Postgres, Tailwind, Server Components — chosen for speed with a small team. Spend novelty on the product. And the people selling exceptional experiences shouldn't use an ugly tool to do it: interface quality is an adoption strategy, and this UI is the ancestor of the client-facing one.

---

## 11. Functional Modules

Thirteen modules. Sequencing at the end.

### M1 — Email Extraction Pipeline *(the front door)*
**Why:** Almost everything the agency needs to know arrives as email. Until that's parsed, every other module is asking people to hand-copy from Gmail.
**Scope:** Connected agency mailboxes read continuously. Classification into known types — enquiry, rate sheet, supplier quote or confirmation, PNR/ticket, passport or ID, visa, payment advice. Field extraction from body and attachments (PDF, image, spreadsheet). Matching to existing family, trip, supplier or property. A confidence score per extraction. Everything output as a **proposal**, never a committed record. Full trace from any record back to the source email.
**Not in scope:** Replacing the mail client, sending outbound mail, auto-replies, WhatsApp or other channels (later, and the pipeline should be channel-shaped so they slot in).
**Later:** The same pipeline serves WhatsApp inbound in year two. Getting the classify → extract → match → propose shape right now is what makes that a channel addition rather than a rebuild.

### M2 — AI Review Queue *(the trust layer)*
**Why:** An AI-first intake is only safe if a human sees everything before it counts. The queue is what turns "the AI might be wrong" from a risk into a measured, bounded, daily-cleared workload.
**Scope:** One queue for every extraction type, so operators learn one interface. Each item shows the proposal beside the source email, with low-confidence fields flagged. Accept / correct / reject in seconds, with keyboard-first review. Routing by type and confidence: low-risk high-confidence items may auto-accept with an audit entry, while anything touching money, identity or rates always needs a person. Bulk handling for rate sheets (dozens of rows from one PDF). Per-type accuracy dashboards built from corrections. Full audit: what was proposed, by which model version, who accepted or changed it.
**Not in scope:** Model training or fine-tuning infrastructure. Corrections are captured as data and as prompt/rule improvements; anything heavier is a later decision.
**Later:** The same queue supervises AI-drafted client replies in year two — an operator approves a WhatsApp response before it sends. **The queue is how the agency earns the confidence to eventually let some of it through unsupervised.**

### M3 — Family, Client & Traveller Management *(highest-leverage module)*
**Why:** The agency sells to households, not individuals. Modelling that properly is what makes memory compound.
**Scope:** **Family** as the primary account — the Mehtas, with lifetime value, relationship owner, tier and history. Members within it, each a **traveller** with their own documents, passport, loyalty numbers and preferences. Roles inside a family (principal contact, payer, minor, elder, PA or family office). Relationships between families for group and multi-generational travel. **Preferences at three levels:** family (always villas, never early flights), person (window seat, gluten-free), and trip-specific. Documents attached to travellers, so a passport collected once is never chased again. Full trip and spend history at family and person level. Merge and dedupe, which will be needed constantly. Corporate and family-office clients handled as a family shape, not a separate entity type.
**Not in scope:** Client-facing profile editing; marketing segmentation.
**Later:** This is the AI assistant's memory. Recommendation quality in year two is capped by preference structure here.

### M4 — Enquiry & Pipeline
**Scope:** Enquiries with source, owner, qualification fields, and stages from new to won/lost with loss reasons. Ageing and follow-up prompts. Pipeline value. Mail-borne enquiries arrive from the review queue pre-filled; other channels entered by hand.
**Not in scope:** Lead scoring, automated nurture.

### M5 — Trip & Itinerary Builder *(core value)*
**Scope:** Trip as the central entity linking family, travellers, dates, destinations and components. Structured components — accommodation, flight, transfer, experience, day segment. Day-by-day composition. Duplicate-and-adapt from past trips, plus a reusable component library. Version history. Cost, sell and margin per component and per trip.
**Not in scope:** Live availability, pricing optimisation, client-side editing.
**Later:** The output target for AI itinerary generation. Structured itineraries can be proposed by an AI; documents can't.

### M6 — Offline Rate Repository *(strategic asset)*
**Why:** Contracted and negotiated rates are the agency's actual edge over public OTAs, and the one thing a client-facing AI could offer that metasearch can't. Today they're PDFs in an inbox.
**Scope:** Property and supplier master data. Contracted rates by room category, season and date range, with validity windows and visible staleness. Inclusions, blackout dates and cancellation terms as structured fields — not a paragraph. Fast search by destination, property, dates and category. Margin shown at selection time. Source contract attached for verification. **Loading is powered by M1/M2** — a rate sheet arrives by mail, is parsed into rows, and Sameer reviews a table instead of transcribing a PDF. Coverage and freshness reported as product metrics.
**Not in scope:** Live or dynamic rates, public-rate comparison, automated contract negotiation.
**Later:** The proprietary inventory layer that makes a client-facing AI differentiated rather than a metasearch skin. **Over five years this is probably the most valuable thing the platform holds.**

### M7 — Sabre Sync Layer
**Why:** Not "a booking feature." Sabre is a second system of record that must agree with ours continuously, and the interesting work is in disagreement, not in copying.
**Scope:** A defined synchronization surface: PNR retrieval, mapping GDS records onto trips and travellers, and a per-record sync state (in sync, diverged, unmatched, stale). Divergence detection with a clear owner and a resolution path — the platform never silently overwrites Sabre and never silently accepts a change it didn't expect. Sync log and last-checked timestamps, so "is this current?" is always answerable. Unmatched PNRs surface in the review queue for a human to attach to the right trip. Ticketing deadlines and schedule changes flow in as tasks. Manual booking entry is always a first-class path, so the agency never stops working if sync is down.
**Not in scope:** Booking or ticketing *into* Sabre from PureLuxe; fare construction; cancellations.
**Later:** The same sync-state pattern extends to supplier and hotel systems. Building it once, properly, for Sabre is what makes the second integration cheap.
**Dependency:** access mechanism and licensing must be validated before this is scoped — see Risks.

### M8 — Bookings & Suppliers
**Scope:** Booking records per component with supplier, reference, status, deadlines and cancellation terms. Hold expiry tracking. Supplier and property records with contacts, terms and contracted status. Requests and replies logged against trips, with inbound mail matched in by the pipeline. Basic supplier performance signals — responsiveness, issue count.
**Not in scope:** Booking into supplier systems; supplier logins or portals.

### M9 — Tasks & Operations
**Scope:** Tasks generated automatically on confirmation, dated from travel dates and destination rules. Assignment, personal and team queues ordered by risk, overdue escalation, per-trip readiness view.
**Not in scope:** General project management; configurable rules engines.
**Later:** The layer AI agents eventually act within — proposing routine actions, and only ever executing under supervision.

### M10 — Document Vault
**Scope:** Documents linked to travellers and trips, typed (passport, visa, insurance, ID), with expiry tracked and validated against travel dates. Access control, full access audit, retention rules, secure request-and-collect. Passport data extracted by M1 and confirmed in M2 — a scan arrives by mail, an operator checks the parsed fields against the image, done.
**Not in scope:** OCR as a hard dependency (manual entry always works); client-facing upload portal.
**Later:** Client-initiated secure upload over WhatsApp — high value early, and it needs this governance to exist first.

### M11 — Trip Money
**Scope:** Cost, sell and margin per component and trip. Payment schedule and milestones, received and outstanding, supplier payment dates. Payment advices from mail matched to trips through the queue.
**Not in scope:** Invoicing, ledgers, tax, reconciliation, accounting integration.

### M12 — Dashboards
**Scope:** Founder view — pipeline and conversion, confirmed revenue and margin, consultant load, trips at risk, document readiness, extraction accuracy and queue health. Consultant view of own book. Ops view of task and document health.
**Not in scope:** Custom report builder, BI tooling, forecasting.

### M13 — Admin & Access
**Scope:** Users, roles, sensitivity-tier permissions, master data (destinations, properties, suppliers, templates), extraction rules and mailbox configuration, audit log, staleness visibility.
**Not in scope:** Custom roles, multi-tenancy.

### Sequencing

**Phase 1 — the spine:** M1, M2, M3. The pipeline, the queue and the family model. Nothing durable gets built on a weak family model, and every later module gets cheaper once intake is parsed.

**Phase 2 — the assets:** M6, M5. Rates first — it's the strategic asset, it proves the pipeline's value fastest, and it's the module consultants feel immediately.

**Phase 3 — operational trust:** M7, M8, M9, M10. Where the agency stops failing clients.

**Phase 4 — visibility:** M4 pipeline reporting, M11, M12.

M4's enquiry capture ships alongside Phase 1 in basic form; its reporting depth comes later. Phases 1–2 must land before broad rollout — a platform without rates and itineraries is asking consultants to do data entry for someone else's benefit, which is the standard way internal tools die.

---

## 12. Future Expansion Strategy

Three phases. The build pays only for cheap optionality, not for the future product.

### Now (0–6 months) — internal source of truth, AI at intake only
Ship M1–M13 in sequence. Hit the adoption numbers. Accumulate the structured data everything else needs. AI appears **only** as extraction behind the review queue — no AI-generated client-facing content, no AI acting without a human.

*Move on only when:* enquiry capture ≥ 95% and leakage < 2%; extraction acceptance ≥ 85% with the queue cleared daily; ≥ 80% of active families have structured preferences; ≥ 90% of frequently sold properties have in-date rates; itineraries are 100% generated from trip data; consultants say the platform is faster than what they used before.

### Next (6–12 months) — AI as internal copilot
Still internal, still reviewed, but now generative rather than only extractive. Natural-language search across families, trips and rates. Draft itineraries from an enquiry brief, for the consultant to edit. Communication summaries and suggested next actions. Anomaly detection for at-risk trips, expiring documents, stale contracts. Drafted supplier and client replies — **queued for approval, not sent.**

Every one has a human in the loop by design. The real purpose of this phase is to find out, with evidence from the queue, whether the data foundation is good enough to ground AI *before* a client ever reads an AI-written word.

### Later (12+ months) — client-facing, WhatsApp-first
Only on the strength of the queue's numbers. WhatsApp becomes the primary client channel, flowing through the same intake shape into the same families and trips. The AI assistant handles qualification, availability questions, itinerary previews and status — with a consultant always one step away, and with everything it says still passing the review queue until accuracy earns it otherwise. Client self-service for document upload, itinerary viewing and payment. Booking assistance automated progressively, under supervision.

The consultant's role shifts from executor to curator and escalation point. **Not removed** — in luxury travel the relationship is the product. AI takes the friction, not the person.

### The six things the build must get right for this to work
Everything else can be deferred. These are cheap now and expensive later:

1. **Stable identity** for families, travellers, trips, bookings, documents, suppliers — something an external channel can reference.
2. **Intake as a channel-agnostic shape.** Classify → extract → match → propose → review. Email is the first channel; WhatsApp must be a second channel, not a second pipeline.
3. **A queue that is the single approval point** for anything machine-generated. Adding a new proposal type later must not mean a new review UI.
4. **Trip and booking state as explicit, queryable data** — answerable by a machine, not inferred from a PDF.
5. **A permission model that can accept the client** as an actor without restructuring.
6. **A durable event and audit trail**, including which model proposed what and which human accepted it. This is the grounding substrate and the accountability record for anything AI later does alone.

**Do not build now:** client auth or portals, WhatsApp integration, payment gateway, model training infrastructure, multi-tenancy, public search, recommendation engines, mobile apps, supplier connectivity.

---

## 13. Assumptions

**Business.** Team stays roughly 10–20 users through the first year — design for consultant leverage, not throughput. Trip volume is low hundreds to low thousands a year, not tens of thousands. Luxury positioning holds; a volume pivot would change the product materially. The founder will mandate use — voluntary adoption of an internal system of record doesn't work. Existing process is broadly sound and should be encoded, not reinvented.

**Data.** Mail volume and structure are consistent enough for classification to work — supplier rate sheets and confirmations follow recognisable patterns per supplier. Mailbox access is available with the right permissions and retention. Historical family and booking data migrates to a useful if imperfect degree; some history will be lost and that's planned for, not discovered. Identity dedupe is achievable with modest manual effort. Rate contracts can be loaded at acceptable cost — **with the pipeline this is review rather than transcription, but it still needs a named owner and a deadline.**

**AI.** Extraction quality on real agency mail reaches ≥ 85% acceptance within a few months of tuning. **This is the assumption the product bets most on, and it should be tested against real sample mail before the pipeline is fully scoped.** Per-item review cost stays low enough that the queue is a minutes-a-day job, not a role.

**Technical.** The stack is sufficient for the first two phases without extra infrastructure. Mostly desktop use in working hours; responsive web is fine. Cloud-hosted, online-only, no offline mode. One workspace.

**Sabre.** Access for retrieval and reconciliation exists through an available, permitted mechanism. **Validate before scoping M7** — largest external dependency in the build.

**Regulatory.** Handling passports creates real obligations (India's DPDP Act; GDPR where EU clients are served). Mail ingestion adds its own — the platform will hold copies of client correspondence. Confirm with counsel **before** M1 and M10 are built, not after. Data residency requirements known and satisfiable.

---

## 14. Risks

**Consultants don't switch.** *The main way this fails.* They keep using WhatsApp and Excel, the platform half-fills, nobody trusts it, which justifies not using it. Happens when it's slower than what it replaces, or when data entry benefits someone else. → Obsess over speed in daily-touch modules; ship consultant value (rates, itineraries) before founder value (dashboards); make Rahul the champion; instrument adoption from week one and treat a drop as urgent; founder mandate plus visible founder use.

**Extraction isn't accurate enough.** If acceptance sits at 60%, the queue becomes a second data-entry job and the whole intake premise collapses. → Test on real sample mail before committing scope; start with the highest-volume, most-patterned types (supplier confirmations, PNRs, rate sheets from known suppliers) rather than the general case; keep manual entry first-class everywhere; publish per-type accuracy so the picture is honest; be willing to narrow which types are automated.

**The queue becomes a backlog.** An unreviewed queue is worse than no queue — records go stale and people route around it. → Design for seconds-per-item, keyboard-first; auto-accept high-confidence low-risk types; alert on backlog growth; give it a named daily owner.

**Rates never reach critical mass.** Partial coverage means consultants still check the old files — so they check the old files, and the module dies. → Resource loading as a funded project with an owner and a date; use the pipeline to make it review not transcription; target the top ~50 sold properties first for fast density; report coverage weekly to leadership.

**Sabre access proves harder than assumed.** Licensing, certification or technical limits could make sync impractical in the timeline. **Higher stakes now that Sabre is a sync layer rather than a booking field.** → Validate before scoping M7; keep manual booking entry a first-class path so nothing blocks; be willing to ship a reduced sync (read-only PNR retrieval, no continuous reconciliation) or defer entirely.

**Scope drifts toward the OTA.** The compelling future pulls the team into client-facing AI before the internal foundation is stable — producing a great demo with no data behind it. → The phase-exit numbers in §12 are gates, not aspirations; the six commitments are the *only* permitted concessions to the future.

**Privacy exposure.** Centralising passports *and* client correspondence creates a high-value target and real obligations. In a luxury client base a breach is existential, not a compliance ticket. → Access control, audit, encryption, retention and least-privilege in v1; counsel before M1 and M10; keep admin access separate from document access.

**Migration pollutes the family record.** Imported history is incomplete or duplicated, consultants lose confidence, and they go back to their own files. → Treat migration as a feature with acceptance criteria; ship merge/dedupe from day one; be explicit about what history is and isn't there; prefer a clean partial dataset to a polluted complete one.

**People fear the AI is aimed at them.** If consultants read "AI extraction" as "headcount reduction," adoption dies quietly and nobody says why. → Say principle 5 out loud, repeatedly; make the first AI wins visibly the work nobody wants — transcription, chasing, checking; let consultants see the queue is theirs to control.

**A wrong number kills the dashboard.** One visibly incorrect figure and the founder never trusts reporting again. → Narrow, well-defined metrics; every number traceable to its records; ship M12 only when its inputs are reliably populated.

**Over-modelling.** Travel is genuinely complex and it's easy to build something elegant, general, and slow to change. → Model this agency, not the industry; no configurability in v1; stay concrete until a second real use case shows up.

---

## 15. Out of Scope

Excluded from this planning horizon — listed so they're raised as changes, not assumed as oversights.

**Client-facing:** login, portal, account, public website or search, client mobile app, direct client payment, client-editable itineraries.

**Channels:** WhatsApp Business integration, outbound mail sending or inbox replacement, telephony and call recording, social media management. *(Inbound mail reading is in scope; everything else about mail is not.)*

**Booking and inventory:** live availability, rate shopping, automated booking into supplier or GDS systems, air ticketing and fare construction, channel management, dynamic packaging.

**Financial:** invoicing, receipting, accounting and ledgers, tax and filing, payroll and commissions, treasury and multi-currency management.

**Organisational:** HR and attendance, multi-tenancy, franchise or white-label, supplier portals, B2B agent distribution.

**Technical:** native apps, offline mode, custom model training or hosting infrastructure, public third-party API, data warehouse or BI platform, internationalisation.

**Process:** configurable workflow engine, custom fields, custom roles, plugin ecosystem.

---

## Appendix — Open Questions for CTO Review

Recorded rather than guessed at; each changes scope or sequencing.

1. **Extraction accuracy on real mail.** Can we get a sample of a few hundred real agency emails and test classification and field extraction before committing M1's scope? This is the product's largest technical bet.
2. **Mailbox access.** Which mailboxes, whose permission, what retention, and what happens to mail that has nothing to do with travel? Privacy scope needs deciding before build.
3. **Auto-accept boundaries.** Which extraction types and confidence levels may bypass human review, and who owns changing that line as accuracy improves?
4. **Sabre access.** What mechanism, licensing and effort does sync actually require — and what's the reduced version if the full one isn't available?
5. **Family model edge cases.** Divorced parents, family offices booking for multiple households, corporate clients with personal trips. Worth an hour with Priya before modelling.
6. **Rate loading ownership.** Who reviews the initial contract load, over what period, to what coverage target? Without a name this risk lands by default.
7. **Privacy obligations.** Which regimes apply given the client base, and what do they require of mail ingestion specifically?
8. **Rollout.** Everyone at once, or pilot with two consultants first? Affects adoption risk and how much early metrics are worth.
9. **Document retention.** How long are passports held after travel? A business decision with direct architectural consequences.
