# Edit Event & Publish Event — Spec vs Backend Gap Analysis

This document compares the **full technical breakdown** (Edit Event, Publish Event, API endpoints, backend flow, data models, key rules) against the **current eventopia-stagging-backend** implementation. It states what is implemented, what is partial, and what is missing.

---

## 1. API path mapping (spec vs actual)

| Spec endpoint | Actual backend | Status |
|---------------|----------------|--------|
| POST /api/events/create | **POST /api/promoters/events** (and **POST /api/v1/events** with tiers) | ✅ Different path, same intent |
| PATCH /api/events/{event_id}/update | **PATCH /api/promoters/events/:eventId** | ✅ Different path, same intent |
| POST /api/events/{event_id}/publish | **POST /api/promoters/events/:eventId/publish** | ✅ Different path, same intent |
| GET /api/events/{event_id} | **GET /api/promoters/events/:eventId** and **GET /api/v1/events/:id** | ✅ Exists |
| POST /api/events/{event_id}/cancel | **POST /api/promoters/events/:eventId/cancel** | ⚠️ Implemented but see “Cancel flow” below |
| POST /api/events/{event_id}/postpone | — | ❌ **Not implemented** |
| GET /api/events/{event_id}/tickets | **GET /api/promoters/events/:eventId/ticket-types** | ✅ Same concept, different name |
| POST /api/events/{event_id}/tiers/create | **POST /api/promoters/events/:eventId/ticket-types** | ✅ Same concept |
| PATCH /api/events/{event_id}/tiers/{tier_id}/update | **PATCH /api/promoters/ticket-types/:ticketTypeId** | ✅ Exists |
| DELETE /api/events/{event_id}/tiers/{tier_id} | **DELETE /api/promoters/ticket-types/:ticketTypeId** | ✅ Exists |
| GET /api/booking-fee/resolve?price={amount} | — | ❌ **No HTTP endpoint** (logic exists in tierResolver.service.js only) |

**Summary:** Create, update, publish, get event, get ticket types, create/update/delete tier are implemented under **/api/promoters/...** (and v1 where applicable). **Postpone** and **GET booking-fee/resolve** are not exposed as in the spec.

---

## 2. What is implemented

### 2.1 Event management

- **Create event (draft):** POST /api/promoters/events (body: title, description, startAt, endAt, format, accessMode, visibilityMode, city, venueName, venueAddress, etc.) and POST /api/v1/events (with ticket_tiers in one call). Returns event id.
- **Update event:** PATCH /api/promoters/events/:eventId. Partial updates allowed. **Restriction:** Cancelled events cannot be edited; there is **no** extra restriction yet for “live + tickets sold” (see gaps).
- **Publish:** POST /api/promoters/events/:eventId/publish. Validation: title, description, start_at, end_at, city, format, access_mode; venue_name/venue_address if in_person/hybrid; at least one active ticket type if ticketed/mixed. Sets status to `published`, published_at. **Does not** create escrow record, generate event_slug, QR, or referral link (see gaps).
- **Get event:** GET /api/promoters/events/:eventId (full object + tags + ticketTypes) and GET /api/v1/events/:id (simplified + ticket_tiers). No escrow_status in response (see data model).
- **Cancel:** POST /api/promoters/events/:eventId/cancel. **Currently sets status directly to `cancelled`**. Spec requires **pending_cancellation** and routing to Promoter Manager first — not implemented.

### 2.2 Ticket tiers

- List: GET /api/promoters/events/:eventId/ticket-types (items + pagination).
- Create: POST /api/promoters/events/:eventId/ticket-types.
- Update: PATCH /api/promoters/ticket-types/:ticketTypeId.
- Delete: DELETE /api/promoters/ticket-types/:ticketTypeId.
- Pause/Resume/Duplicate: POST endpoints exist.
- **Booking fee:** Stored in ticket_types (booking_fee_amount). For v1 create it is **server-calculated** via tierResolver (promoter cannot override). For legacy ticket-type create/update, backend can set it; no explicit “block client override” documented in one place (see key rules).

### 2.3 Booking fee resolution (logic only)

- **tierResolver.service.js** has `resolveTier(ticketPrice)` and `getBookingFeePence(ticketPricePounds)` (tier bands, booking fee, distributable pool). Used server-side in v1 event create and ticket purchase.
- **No GET /api/booking-fee/resolve?price=...** endpoint. Frontend cannot “call live as promoter types ticket price” via this API.

### 2.4 Audit

- **event_audit_logs** table exists (event_id, promoter_id, action, field_name, old_value, new_value, request_id, ip_address, user_agent, created_at). **No** JSON “changed_fields” delta; single field_name/old_value/new_value per row.
- **logEventChange()** is called on publish, cancel, pause, image delete, etc. Logs are **insert-only** (no update/delete on the log table) — aligns with “immutable” rule.

### 2.5 Escrow and credit

- **Escrow:** Territory-level escrow_accounts; credit on **purchase** (escrowReceive + allocateCredit). **No** “create escrow liability record at publish time with initial value 0” — escrow is used when tickets are sold.
- **Credit:** Projected credit is written on purchase; “display-only until event concluded” is a business rule (no API change needed). No “575 ticket threshold” or “sprint/unlock” in event publish flow.

---

## 3. What is partial or different

### 3.1 Publish validation (pre-publish checklist)

- **Implemented:** title, description, start_at, end_at, city, format, access_mode; venue_name/venue_address for in_person/hybrid; at least one active ticket type for ticketed/mixed.
- **Not implemented in backend:**  
  - End date > start date (can be added).  
  - “Venue **or** stream link present” (stream_url not in publish check).  
  - Banner image uploaded (cover_image_url not required at publish).  
  - Age restriction confirmed.  
  - Attendee name collection confirmed for multi-ticket.  
  - Promoter bank details on file.  
  - 575 ticket threshold / unlocked status.

So the **pre-publish checklist** from the spec is only partly reflected in the API; several checks are missing.

### 3.2 Update restrictions when event is live and has tickets sold

- Spec: when status is live and tickets sold, restrict changes to description, banner, end time (within rules), stop sale toggle.
- Current: only “cancelled events cannot be edited”. **No** restriction that “when published and tickets_sold > 0, only certain fields are updatable”. So **not** fully implemented.

### 3.3 Ticket price change after first sale

- Spec: “Ticket price changes after first sale are blocked at API level.”
- Current: **Not** enforced in PATCH /api/promoters/ticket-types/:ticketTypeId. Backend does not check qty_sold and block price/booking_fee updates. So **not** implemented.

### 3.4 Cancel flow (pending_cancellation)

- Spec: Cancel must route through **pending_cancellation** and “Promoter Manager queue”; does not publish cancellation publicly immediately.
- Current: Cancel sets status directly to **cancelled** and records cancel_reason. **No** pending_cancellation status or queue. So **not** implemented.

### 3.5 Post-publish artefacts

- Spec: On publish — generate event_slug, QR code, referral tracking link (tied to promoter_id and guru_id), create escrow liability record, post to discovery.
- Current: Only status → published and published_at. **No** event_slug, **no** QR generation at publish, **no** event-level referral link at publish (guru referral exists for signup, not “event URL + guru”). **No** escrow liability record at publish. Discovery listing exists for public events but is not explicitly “post to discovery index” in publish handler.

---

## 4. What is missing (data model and fields)

### 4.1 Events table

- Spec mentions: doors_open_time, is_online, stream_url, banner_image_url, age_restriction, max_tickets_per_order, collect_attendee_names, stop_sale_at, type (public/private/invitation only).
- Current: **doors_open_time** — no; **stream_url** — no (meeting_link, live_access_url, ondemand_access_url exist for other use); **banner** — cover_image_url used; **age_restriction** — no; **max_tickets_per_order** — on ticket_types (per_order_limit), not event-level; **collect_attendee_names** — not on events (purchase API accepts attendee_names); **stop_sale_at** — no; **type** — visibility (public/private_link) exists, no “invitation only”. So several **UI-driven fields** are missing in the events table.

### 4.2 Ticket tiers table

- Spec: face_value, booking_fee, vat_amount, noda_fee, distributable_pool, sales_start, sales_end, is_visible.
- Current: price_amount, booking_fee_amount, sales_start_at, sales_end_at, visibility, capacity_total, qty_sold. **No** vat_amount, noda_fee, distributable_pool as columns (tierResolver computes distributable_pool in code only). Naming differs (face_value vs price_amount) but concept is there.

### 4.3 Escrow ledger (per-event)

- Spec: Escrow Ledger with event_id, promoter_id, territory_id, total_ticket_revenue, booking_fee_collected, escrow_balance, status (holding/released/refunded).
- Current: escrow_accounts are **per territory**, not per event. No “escrow ledger row per event” with holding/released/refunded. So **not** as in spec.

---

## 5. Key rules — compliance

| Rule | Spec | Backend |
|------|------|--------|
| Booking fee read-only for promoter | Yes | ✅ v1: server-calculated only. Legacy ticket-type API: could allow override unless explicitly blocked. |
| Credit projections display-only until concluded | Yes | ✅ No API pays out credit before conclusion; display-only is frontend/business rule. |
| Block ticket price change after first sale | Yes | ❌ Not enforced in PATCH ticket-type. |
| Cancellation via pending_cancellation only | Yes | ❌ Direct to cancelled. |
| Escrow entry at publish time | Yes | ❌ Escrow used on sale, not “create liability at publish”. |
| Attendee names enforced when collect_attendee_names and order > 1 ticket | Yes | ⚠️ Purchase API requires attendee_names length = quantity; no event-level “collect_attendee_names” flag. |
| Audit log immutable | Yes | ✅ Insert-only; no update/delete. |
| Referral link tied to guru at publish | Yes | ❌ No event-specific referral link generated at publish. |

---

## 6. Summary: are the APIs and functionality fully there?

- **Edit Event:**  
  - **APIs:** Update event and ticket tiers exist (PATCH event, PATCH ticket-type, etc.).  
  - **Functionality:** Partial. Many **UI fields** (doors open, stream URL, age restriction, stop_sale_at, collect_attendee_names, etc.) are not in the backend or not validated. **Restrictions** for “live + tickets sold” (only description, banner, end time, stop sale) and **block ticket price change after sale** are **not** implemented.

- **Publish Event:**  
  - **APIs:** Publish endpoint exists and runs validation.  
  - **Functionality:** Partial. **Pre-publish checklist** is only partly implemented (no banner, no stream/venue either-or, no bank details, no 575 threshold, no attendee name collection check). **Post-publish** (event_slug, QR, event referral link, escrow liability at publish) is **not** implemented.

- **Booking fee:**  
  - **Logic:** Exists (tierResolver). **Read-only** for promoter in v1 flow.  
  - **API:** **GET /api/booking-fee/resolve?price=** is **not** present; frontend cannot resolve fee “live” via API.

- **Cancel / Postpone:**  
  - Cancel: implemented but goes **directly to cancelled** (no pending_cancellation or queue).  
  - Postpone: **no** endpoint.

- **Data model:**  
  - Events and ticket_types are broadly aligned for core fields; **escrow** is territory-level, not per-event. Several spec fields (doors_open, age_restriction, stop_sale_at, collect_attendee_names, etc.) are missing.

So: **the core “edit and publish” APIs exist and work**, but the **full functionality** (pre-publish checklist, post-publish artefacts, update restrictions when live, cancel flow, booking-fee resolve API, and several data model fields) is **not** fully fulfilled. The doc above can be used as a checklist to implement the missing pieces.
