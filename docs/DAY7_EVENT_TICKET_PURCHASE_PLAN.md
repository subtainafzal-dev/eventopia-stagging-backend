# Day 7 — Event & Ticket Purchase Flow: Implementation Plan (No Payment)

**Source:** eventopia_day7_roadmap.docx  
**Scope:** Backend only, **without** GoCardless/payment provider integration. Payment flow is stubbed or skipped so purchase can be tested end-to-end.

---

## 1. What the roadmap describes

- **Roles:** Buyer, Promoter.
- **Goal:** Buyer browses events, selects ticket tiers, sees booking fee, enters attendee names, completes purchase. Promoter creates events with multiple ticket tiers (booking fee system-calculated), sees events with tickets sold, escrow revenue, payout status.
- **Payment (deferred):** GoCardless Instant Bank Pay — create billing request, redirect, webhook confirms → Escrow Receive Service → tickets + credit allocation. **Not implemented in this plan.**

---

## 2. Database models (roadmap vs current)

### 2.1 Event model

| Roadmap (Day 7) | Current codebase |
|-----------------|------------------|
| `event_id` (UUID PK) | `events.id` BIGINT |
| `promoter_id`, `title`, `description`, `event_date`, `venue`, `status` (draft \| pending_review \| live \| cancelled \| completed), `territory_id` | **events** table: `id`, `promoter_id`, `title`, `description`, `start_at`/`end_at`, `venue_name`, `status` (`event_status_enum`: draft, **published**, completed, cancelled, unpublished). **No `pending_review`.** |
| `event_date` (ISO8601) | Use `start_at` as event date |

**Already there:** events table with promoter_id, title, description, start_at, end_at, venue_name, status, territory_id, tickets_sold, etc.  
**Gap:** Status enum has `published` not `live`; no `pending_review`. Map roadmap “live” → “published”, and optionally add “pending_review” or treat “draft → submit → publish” as review step.

### 2.2 Ticket tier model

| Roadmap | Current |
|--------|---------|
| `ticket_tiers`: tier_id, event_id, tier_name, ticket_price, **booking_fee (system-calculated)**, quantity_available, quantity_sold | **ticket_types**: id, event_id, name, price_amount, **booking_fee_amount** (stored, can be manual) |
| booking_fee never supplied by promoter; from Tier Resolver | booking_fee_amount exists but can be set manually |

**Already there:** ticket_types ≈ ticket_tiers (name, price, booking_fee, capacity, qty_sold).  
**Gap:** Day 7 requires booking_fee to be **always** system-calculated via Tier Resolver. So on create/update of a tier, backend must call Tier Resolver and set booking_fee; promoter must not send booking_fee.

### 2.3 Ticket (purchase record) model

| Roadmap | Current |
|--------|---------|
| `tickets`: ticket_id, event_id, buyer_id, tier_id, attendee_name, status (active \| cancelled \| refunded), confirmation_reference, purchased_at, ticket_price, booking_fee | **tickets**: id, order_item_id, event_id, ticket_type_id, ticket_code, buyer_name, status, order_id, user_id, etc. |
| One row per ticket; confirmation_reference same for one order | order_items + tickets; order has order_number; tickets linked to order_item |

**Already there:** orders, order_items, tickets with event_id, ticket_type_id, buyer (user_id), status, etc.  
**Gap:** Response shape for “my tickets” (ticket_id, event_title, event_date, venue, tier_name, ticket_price, booking_fee, attendee_name, status, confirmation_reference, purchased_at). May already be satisfiable from existing tables with different field names (e.g. order_number as confirmation_reference).

### 2.4 Escrow

| Roadmap | Current |
|--------|---------|
| Escrow Receive: ticket_price × qty → escrow; booking_fee × qty → operating account | **escrow_accounts** in migrations (territory_id, balance, pending_liabilities) |

**Already there:** escrow_accounts table.  
**Gap:** No “Escrow Receive” service that credits territory escrow and operating account and writes ledger. Needed when a purchase is confirmed (in no-payment flow: on “purchase” API success; with payment: on webhook).

---

## 3. Services (roadmap vs current)

### 3.1 Tier Resolver — **NOT implemented**

- **Purpose:** Map `ticket_price` → tier label (1–6), `booking_fee`, `distributable_pool`.
- **Bands (roadmap):**  
  Tier 1: £0.01–£5.99 → £1.85 | Tier 2: £6–£15.99 → £2.35 | Tier 3: £16–£29.99 → £2.85 | Tier 4: £30–£60 → £3.50 | Tier 5: £60.01–£99.99 → £4.50 | Tier 6: £100+ → £5.85
- **Used by:** Create event (validate and store booking_fee per tier), GET event detail (append booking_fee), Purchase (confirm fee).
- **Needed:** New service module, e.g. `tierResolver.service.js`, with one function: `resolveTier(ticketPrice)` → `{ tier_label, booking_fee, distributable_pool }`. Throw INVALID_TICKET_PRICE for ≤ 0 or non-numeric.

### 3.2 Escrow Receive Service — **NOT implemented**

- **Purpose:** On confirmed payment: ticket_price × quantity → escrow; booking_fee × quantity → operating account; write ledger.
- **Without payment:** Call this from POST purchase when “confirming” the order immediately (no webhook). Inputs: event_id, territory_id, ticket_price, booking_fee, quantity, buyer_id, order_id (or similar reference).
- **Needed:** New service that: (1) updates escrow_accounts.balance for territory, (2) updates operating account (if table exists) or ledger only, (3) calls `createLedgerEntry()` for audit. Credit allocation is **not** triggered here (roadmap: credit stays projected; escrow not touched until event settles — but for Day 7 “purchase flow” we may still project credit on purchase).

### 3.3 Credit Allocation (allocateCredit) — **Partially / different**

- **Purpose:** Create projected credit for Promoter, Guru, Network Manager per ticket purchase (tier-based splits).
- **Current:** credit_ledger table and possibly platform_ledger/ledger_allocations exist; no single `allocateCredit(event_id, tier_label, quantity, promoter_id, guru_id, network_manager_id)` that writes projected credits and ledger entries.
- **Needed:** Implement allocateCredit that: resolves tier to per-ticket splits (e.g. Tier 1: Promoter £0.50, Guru £0.30, Network £0.10, Eventopia £0.29), multiplies by quantity, writes to credit_ledger (or credit_wallets) with status PROJECTED, and calls `createLedgerEntry()` for each role (audit). Refunded tickets: set credit to VOID, decrement counters.

### 3.4 Ledger entry service — **Implemented**

- **Current:** `createLedgerEntry()` in `ledgerCore.service.js`; immutable ledger_entries table with trigger. Use this for all financial audit entries (escrow receive, credit allocation, refunds, etc.).

---

## 4. API contracts (roadmap vs current)

### 4.1 POST /api/v1/events (Create Event)

- **Roadmap:** Promoter only. Body: title, description, event_date, venue, ticket_tiers[{ tier_name, ticket_price, quantity_available }], age_restriction, event_type. Tier Resolver validates ticket_price and stores booking_fee. Response: event_id, status draft, created_at.
- **Current:** POST `/api/promoter/events` exists (createEvent). Likely accepts different shape (e.g. start_at, ticket types with possible manual booking_fee).
- **Needed (without changing existing):** Either (a) add **new** POST `/api/v1/events` that uses Tier Resolver and writes to existing events + ticket_types, or (b) adapt existing createEvent to use Tier Resolver and forbid client-set booking_fee. Same for response (event_id, status: draft).

### 4.2 GET /api/v1/events/:id (Event detail)

- **Roadmap:** Public for live events; promoter JWT for draft. Response includes ticket_tiers with tier_id, tier_name, ticket_price, **booking_fee**, total_cost (= ticket_price + booking_fee). Tier Resolver used to append booking_fee in real time.
- **Current:** GET `/api/events/:id` (getEventDetail). Returns event and ticket types; may already include price and booking_fee from DB.
- **Needed:** Ensure each tier in response has booking_fee (from Tier Resolver if not stored, or from DB if always set by Tier Resolver on create). Add total_cost. Access: draft only for owner.

### 4.3 GET /api/v1/events/my (Promoter my events)

- **Roadmap:** Promoter only. Query: status (draft \| live \| completed \| cancelled), page, limit. Response: events[{ event_id, title, status, event_date, tickets_sold, gross_revenue_in_escrow, payout_status }], total.
- **Current:** Promoter dashboard or list events may exist under different path (e.g. `/api/promoter/events` or similar). Need list scoped to promoter_id with aggregates (tickets_sold from ticket_types.qty_sold or tickets count; escrow from escrow balance or ledger; payout_status from settlement).
- **Needed:** Implement or align GET `/api/v1/events/my` with promoter_id = caller, aggregate tickets_sold and escrow revenue, return payout_status (e.g. pending/scheduled/paid from event or settlement table).

### 4.4 POST /api/v1/tickets/purchase (Purchase tickets)

- **Roadmap:** Buyer only. Body: event_id, ticket_tier (tier_id), quantity, attendee_names[]. Validation: event live, quantity available, attendee_names.length === quantity. Then: create billing request (GoCardless) → redirect → webhook → Escrow Receive → create tickets → allocateCredit → ledger. Response: confirmation_reference, ticket_ids, total_paid, booking_fee_total, tickets[].
- **Without payment:** Skip GoCardless. Flow: (1) Validate event is live (published). (2) Validate quantity and attendee_names length. (3) Reserve or decrement quantity atomically. (4) Create order in “confirmed” state (or payment_pending then immediately mark confirmed). (5) Create order_items and tickets (one per attendee). (6) Call Escrow Receive (stub: update escrow + operating, write ledger). (7) Call allocateCredit (projected). (8) Return confirmation_reference (e.g. order_number), ticket_ids, totals, tickets[].
- **Current:** POST `/api/orders/events/:eventId/orders` (createOrder) and likely checkout flow. May use idempotency, reservation, payment_intent.
- **Needed:** Either new POST `/api/v1/tickets/purchase` that does the above without payment, or extend existing order flow with a “mock confirm” path (e.g. for testing) that runs Escrow Receive + allocateCredit and marks order/tickets confirmed.

### 4.5 GET /api/v1/tickets/my (Buyer my tickets)

- **Roadmap:** Buyer only. Query: status (active \| cancelled \| refunded), page, limit. Response: tickets[{ ticket_id, event_id, event_title, event_date, venue, tier_name, ticket_price, booking_fee, attendee_name, status, confirmation_reference, purchased_at }], total.
- **Current:** GET `/api/orders/me/tickets` likely returns user’s tickets with event/ticket type info.
- **Needed:** Align response shape with roadmap (field names and structure) or add GET `/api/v1/tickets/my` that returns the same structure from existing tickets/orders/events/ticket_types.

---

## 5. What is already implemented

- **Tables:** events, ticket_types, orders, order_items, tickets, escrow_accounts, credit_ledger, ledger_entries (with immutability trigger), territories, users, promoter/guru/network links.
- **Auth:** requireAuth, requireRole, buyer/promoter flows.
- **Ledger:** createLedgerEntry() and GET /api/v1/ledger, GET /api/v1/ledger/:id, GET /api/v1/ledger/export (King’s Account).
- **Events:** Create event (POST /api/promoter/events), get event detail (GET /api/events/:id), list public events (GET /api/events). Event status: draft, published, completed, cancelled, unpublished.
- **Orders/Tickets:** Create order, get my orders, get my tickets, checkout (payment flow may be Stripe or placeholder). Tickets linked to orders and order_items.
- **No Tier Resolver;** no Escrow Receive service; no allocateCredit in the Day 7 shape; no v1 event/ticket API surface that matches the roadmap exactly.

---

## 6. What is needed (without payment integration)

1. **Tier Resolver service**  
   - Implement tier bands and booking fees (Tiers 1–6).  
   - `resolveTier(ticketPrice)` → tier_label, booking_fee, distributable_pool.  
   - Used on event create, event detail, and purchase.

2. **Event create (v1 or align)**  
   - Accept ticket_tiers with tier_name and ticket_price only (no booking_fee from client).  
   - For each tier call Tier Resolver; store booking_fee on ticket_types (or equivalent).  
   - Save event as draft. Optionally support “submit for review” (pending_review) if desired.

3. **Event detail (v1 or align)**  
   - Return each tier with ticket_price, booking_fee (from DB or Tier Resolver), total_cost.  
   - Draft: only owner can read.

4. **Promoter my events (v1)**  
   - GET list by promoter_id; include tickets_sold, gross_revenue_in_escrow, payout_status.  
   - Use existing events + ticket_types + escrow/settlement data.

5. **Escrow Receive service**  
   - Inputs: event_id, territory_id, ticket_price, booking_fee, quantity, buyer_id, order reference.  
   - Update escrow_accounts.balance (and operating account if applicable); call createLedgerEntry() for audit.  
   - No GoCardless; called from purchase flow when order is confirmed.

6. **Credit Allocation (allocateCredit)**  
   - Inputs: event_id, tier_label, quantity, promoter_id, guru_id, network_manager_id (from event).  
   - Resolve tier splits; write projected credit (credit_ledger or credit_wallets); call createLedgerEntry() per role.  
   - Handle refunds: VOID credit, decrement counters.

7. **Purchase flow (no payment)**  
   - Validate: event live (published), quantity available, attendee_names.length === quantity.  
   - Create order + order_items + tickets (one per attendee).  
   - Call Escrow Receive; call allocateCredit; return confirmation_reference, ticket_ids, totals, tickets[].

8. **My tickets (v1 or align)**  
   - Response: ticket_id, event_id, event_title, event_date, venue, tier_name, ticket_price, booking_fee, attendee_name, status, confirmation_reference, purchased_at.

9. **Optional:** Add `pending_review` to event status enum and a “submit for review” step before “publish” (live) if product wants it.

---

## 7. Suggested implementation order (no code changes here)

1. **Tier Resolver** — New service; unit tests for bands and INVALID_TICKET_PRICE.
2. **Escrow Receive** — New service; uses existing escrow_accounts and createLedgerEntry(); test with a stub purchase.
3. **allocateCredit** — New service; uses credit_ledger/credit_wallets and createLedgerEntry(); depends on tier splits (from Tier Resolver or separate config).
4. **Event create (v1)** — New or refactor: validate tiers with Tier Resolver, store booking_fee, save event + ticket_types as draft.
5. **Event detail (v1)** — Ensure booking_fee and total_cost per tier; draft access only for owner.
6. **Promoter my events (v1)** — List with tickets_sold, escrow revenue, payout_status.
7. **Purchase API (no payment)** — POST /api/v1/tickets/purchase: validate → create order/tickets → Escrow Receive → allocateCredit → return response.
8. **My tickets (v1)** — GET /api/v1/tickets/my with roadmap response shape.

Route strategy: either add all under `/api/v1/` (events, tickets) and keep existing `/api/events` and `/api/orders` unchanged, or align existing routes to the same behaviour and add only missing pieces. Documentation and Postman can target v1 for Day 7.

---

## 8. Out of scope in this plan

- GoCardless (or any real payment provider): billing request, redirect, webhook, signature verification.
- Frontend: Buyer/Promoter screens are described in the roadmap but not implemented in backend.
- Refund flow: mentioned (VOID credit, decrement) but not designed in detail.
- Payout execution (Promoter payout): mentioned in roadmap for later; not part of this plan.

---

*This plan is for implementation only; it does not modify any existing code or migrations.*
