# Promoter Event APIs — Detail and Flow

All promoter event APIs require **authentication** (Bearer token) and an **active promoter** account.  
Base path for promoter routes: **/api/promoters**.  
There is also a **v1** set under **/api/v1/events** (create, my list, my export, detail).

---

## 1. Two ways to manage events

- **Legacy / full flow:** `/api/promoters/events` — create event (no ticket tiers in body), then add ticket types, images, category, tags, then publish. Full control over all fields.
- **v1 / Day 7 flow:** `/api/v1/events` — create event with **ticket_tiers** in one request (tier name, price in GBP, quantity). Simpler for “create event + tiers” in one step. Then use promoter image/category/tags/publish as needed.

You can use either; the document below covers **both** and the **shared** promoter-only APIs (images, status, tickets, etc.).

---

## 2. API list (promoter event–related)

### 2.1 Event list and create

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/promoters/events | List promoter’s events (all). |
| POST | /api/promoters/events | Create event (draft). Body: title, description, startAt, endAt, timezone, format, accessMode, visibilityMode, city, venueName, venueAddress, lat, lng, categoryId, tagIds, tagNames. |
| GET | /api/v1/events/my | List promoter’s events with **search**, **sort**, **filter** (status), **pagination**. Query: status, search or q, sort (soonest \| oldest \| title_asc \| title_desc \| tickets_sold), page, limit. |
| GET | /api/v1/events/my/export | Download promoter’s events as **CSV**. Same query params as “my” (status, search, sort); no pagination. |
| POST | /api/v1/events | Create event (draft) **with ticket tiers in one request**. Body: title, event_date, description, venue, ticket_tiers[] (tier_name, ticket_price in GBP, quantity_available). |

### 2.2 Single event (detail, update, delete)

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/promoters/events/:eventId | Get one event detail (promoter owner only). |
| GET | /api/v1/events/:id | Get event detail (public if published; draft only if owner). Optional auth. |
| PATCH | /api/promoters/events/:eventId | Update event. Body: any of title, description, startAt, endAt, timezone, format, accessMode, visibilityMode, city, venueName, venueAddress, lat, lng, categoryId, etc. |
| DELETE | /api/promoters/events/:eventId | Delete event (promoter owner only). |

### 2.3 Images (cover and gallery)

| Method | Path | Purpose |
|--------|------|---------|
| POST | /api/promoters/events/:eventId/images?type=cover | Upload **cover** image. Body: multipart/form-data, field name **image** (file). |
| POST | /api/promoters/events/:eventId/images?type=gallery | Upload **gallery** image (max 10 per event). Body: same, field **image**. |
| DELETE | /api/promoters/events/:eventId/cover | Remove cover image. |
| DELETE | /api/promoters/events/:eventId/gallery/:imageId | Remove one gallery image (imageId = event_media.id). |
| PATCH | /api/promoters/events/:eventId/images/reorder | Reorder gallery. Body: { "imageOrder": [id1, id2, ...] }. |

### 2.4 Category and tags

| Method | Path | Purpose |
|--------|------|---------|
| PUT | /api/promoters/events/:eventId/category | Set category. Body: { "categoryId": number \| null }. |
| PUT | /api/promoters/events/:eventId/tags | Set tags. Body: { "tagIds": number[], "tagNames": string[] } (optional). |

### 2.5 Status lifecycle (publish, pause, cancel, republish, complete)

| Method | Path | Purpose |
|--------|------|---------|
| POST | /api/promoters/events/:eventId/publish | Set status to **published** (validates required fields and at least one active ticket type). |
| POST | /api/promoters/events/:eventId/pause | Pause event (e.g. status change; exact behaviour depends on backend). |
| POST | /api/promoters/events/:eventId/cancel | Cancel event. |
| POST | /api/promoters/events/:eventId/republish | Republish a paused/unpublished event. |
| POST | /api/promoters/events/:eventId/complete | Mark event completed. |

### 2.6 Ticket types (per event)

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/promoters/events/:eventId/ticket-types | List ticket types for the event. |
| POST | /api/promoters/events/:eventId/ticket-types | Create a ticket type for the event. |
| PATCH | /api/promoters/ticket-types/:ticketTypeId | Update a ticket type (ownership checked). |
| POST | /api/promoters/ticket-types/:ticketTypeId/duplicate | Duplicate a ticket type. |
| POST | /api/promoters/ticket-types/:ticketTypeId/pause | Pause ticket type. |
| POST | /api/promoters/ticket-types/:ticketTypeId/resume | Resume ticket type. |
| DELETE | /api/promoters/ticket-types/:ticketTypeId | Delete a ticket type. |

### 2.7 Event performance and operations

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/promoters/events/:eventId/performance | Get performance/metrics for the event. |
| GET | /api/promoters/events/:eventId/attendees | List attendees for the event. |
| POST | /api/promoters/events/:eventId/validate | Validate a ticket (scanner). |
| POST | /api/promoters/events/:eventId/checkin | Check-in a ticket. |
| POST | /api/promoters/events/:eventId/checkin/undo | Undo check-in. |
| GET | /api/promoters/events/:eventId/logs | Get event audit/logs. |
| GET | /api/promoters/events/:eventId/access-settings | Get access settings. |
| PATCH | /api/promoters/events/:eventId/access-settings | Update access settings. |
| POST | /api/promoters/events/:eventId/rotate-access | Rotate access links. |

---

## 3. Flow (step by step)

### 3.1 Create event and get it live (legacy flow)

1. **Create event (draft)**  
   **POST /api/promoters/events**  
   Body: title, description, startAt, endAt, timezone, format, accessMode, visibilityMode, city, venueName, venueAddress, lat, lng, categoryId, tagIds, tagNames (as needed).  
   → Returns event id.

2. **Add ticket types**  
   **POST /api/promoters/events/:eventId/ticket-types**  
   (Repeat for each tier or use v1 create with tiers in one go.)

3. **Upload images**  
   **POST /api/promoters/events/:eventId/images?type=cover** with form field **image** (file).  
   **POST /api/promoters/events/:eventId/images?type=gallery** with form field **image** (up to 10 times).

4. **Set category and tags (optional)**  
   **PUT /api/promoters/events/:eventId/category** with { categoryId }.  
   **PUT /api/promoters/events/:eventId/tags** with { tagIds, tagNames }.

5. **Publish**  
   **POST /api/promoters/events/:eventId/publish**  
   Backend checks required fields and at least one active ticket type; then sets status to **published**.

After that, the event is live for buyers (if they use an API that lists published events).

### 3.2 Create event with tiers in one go (v1 flow)

1. **Create event with tiers**  
   **POST /api/v1/events**  
   Body: title, event_date, description, venue, ticket_tiers[] (each: tier_name, ticket_price in GBP, quantity_available).  
   → Event is created as **draft** with ticket_types already created.

2. **Images, category, tags (optional)**  
   Same as above:  
   POST /api/promoters/events/:eventId/images (cover/gallery),  
   PUT category, PUT tags.

3. **Publish**  
   **POST /api/promoters/events/:eventId/publish**  
   (Same as legacy; event must meet publish validation.)

### 3.3 Dashboard: list, search, sort, filter, download

1. **List with filters**  
   **GET /api/v1/events/my?status=published&search=...&sort=soonest&page=1&limit=20**  
   Use for “Live events” table: status filter (ALL STATUS), search (Q Search events), sort (SORT: SOONEST, etc.), pagination.

2. **Download CSV**  
   **GET /api/v1/events/my/export?status=...&search=...&sort=...**  
   Same filters as list; response is CSV file (e.g. trigger from download icon).

### 3.4 Edit and lifecycle after create

- **Update event:** PATCH /api/promoters/events/:eventId (body: fields to change).  
- **Pause:** POST .../pause.  
- **Cancel:** POST .../cancel.  
- **Republish:** POST .../republish.  
- **Complete:** POST .../complete.  
- **Delete event:** DELETE /api/promoters/events/:eventId.

### 3.5 Images after create

- **Add cover:** POST .../images?type=cover, body field **image**.  
- **Add gallery:** POST .../images?type=gallery, body field **image** (max 10).  
- **Delete cover:** DELETE .../cover.  
- **Delete gallery image:** DELETE .../gallery/:imageId.  
- **Reorder gallery:** PATCH .../images/reorder, body { imageOrder: [id1, id2, ...] }.

---

## 4. Per-API detail (what to send and what you get)

### POST /api/promoters/events (create event)

- **Auth:** Required; active promoter.
- **Body (JSON):**  
  title, description, startAt, endAt, timezone (default Europe/London), format (e.g. in_person, online_live, hybrid), accessMode (e.g. ticketed), visibilityMode (e.g. public, private_link), city, venueName, venueAddress, lat, lng, categoryId, tagIds (array), tagNames (array).  
  At least one of title, description, city, startAt, endAt required.
- **Response:** Event id (and possibly full event object depending on implementation).
- **Flow:** Creates draft; you then add ticket types, images, then publish.

### POST /api/v1/events (create event with tiers)

- **Auth:** Required; promoter.
- **Body (JSON):**  
  title, event_date (ISO), description, venue, ticket_tiers[] — each tier: tier_name, ticket_price (GBP), quantity_available.
- **Response:** event_id, status draft, title, created_at.
- **Flow:** One step to get event + ticket types; then images/category/tags/publish via promoter APIs.

### GET /api/promoters/events (list my events)

- **Auth:** Required; active promoter.
- **Query:** Optional pagination/filters (implementation-dependent).
- **Response:** Array of promoter’s events (summary fields).

### GET /api/v1/events/my (list with search, sort, filter)

- **Auth:** Required; promoter.
- **Query:** status (draft \| published \| live \| completed \| cancelled), search or q (title search), sort (soonest \| oldest \| title_asc \| title_desc \| tickets_sold), page, limit.
- **Response:** { events: [...], total }. Each event: event_id, title, status, event_date, tickets_sold, gross_revenue_in_escrow, payout_status.

### GET /api/v1/events/my/export (download CSV)

- **Auth:** Required; promoter.
- **Query:** Same as “my” (status, search, sort); no page/limit.
- **Response:** CSV file; header X-Export-Rows with row count. Filename e.g. my_events_export.csv.

### GET /api/promoters/events/:eventId (event detail for promoter)

- **Auth:** Required; event must belong to promoter.
- **Response:** Full event object (all fields promoter needs to edit or view).

### GET /api/v1/events/:id (event detail public/draft-owner)

- **Auth:** Optional. If draft, only owner can see.
- **Response:** event_id, title, description, event_date, venue, status, promoter_id, escrow_protected, ticket_tiers[] (tier_id, tier_name, ticket_price, booking_fee, total_cost, quantity_available, quantity_sold).

### PATCH /api/promoters/events/:eventId (update event)

- **Auth:** Required; event owner.
- **Body (JSON):** Any subset of title, description, startAt, endAt, timezone, format, accessMode, visibilityMode, city, venueName, venueAddress, lat, lng, categoryId, tagIds, tagNames, etc.
- **Response:** Success and/or updated event.

### POST /api/promoters/events/:eventId/images (upload image)

- **Auth:** Required; event owner.
- **Query:** type=cover or type=gallery.
- **Body:** multipart/form-data; field name **image** (file). Allowed: image types (e.g. JPEG, PNG, GIF, WebP).
- **Response (cover):** mediaId, coverImageUrl (e.g. /uploads/filename).  
  **Response (gallery):** mediaId, imageUrl, sortOrder.  
  Full URL = BACKEND_URL + returned path (e.g. https://api.example.com/uploads/photo-123.jpg).

### POST /api/promoters/events/:eventId/publish (publish event)

- **Auth:** Required; event owner.
- **Body:** None (or empty).
- **Checks:** Title, description, start/end, city, format, accessMode; if in_person/hybrid then venueName, venueAddress; if ticketed/mixed then at least one active ticket type.
- **Response:** id, status published, publishedAt.
- **Errors:** 400 if validation fails (missing required fields or ticket types).

### Ticket types (summary)

- **GET /api/promoters/events/:eventId/ticket-types** — list.  
- **POST /api/promoters/events/:eventId/ticket-types** — create (body: name, price, capacity, etc. as per backend).  
- **PATCH /api/promoters/ticket-types/:ticketTypeId** — update.  
- **DELETE /api/promoters/ticket-types/:ticketTypeId** — delete.  
- Pause/resume/duplicate use the routes in the table above.

---

## 5. GET response shapes and how to show them

All successful responses are wrapped as: **`{ "error": false, "data": <payload>, "meta": { "requestId", "timestamp" } }`**. Below, **“Response”** means the **`data`** object. Use it to drive your UI (tables, cards, detail views, badges).

---

### GET /api/promoters/events (list events)

**Query:** status, sort (updated | created | start | title | tickets), page, pageSize, search.

**Response (data):**
```json
{
  "items": [
    {
      "id": 27,
      "title": "My First Event",
      "status": "draft",
      "visibility_mode": "public",
      "startAt": "2026-03-03T11:19:00.000Z",
      "endAt": "2026-03-03T11:19:00.000Z",
      "city": "London",
      "venueName": "The Venue",
      "format": "in_person",
      "access_mode": "ticketed",
      "coverImageUrl": "/uploads/cover-123.jpg",
      "galleryImageUrls": ["/uploads/g1.jpg"],
      "tickets_sold": 2,
      "capacityTotal": 100,
      "views_count": 0,
      "publishedAt": null,
      "share_token": null
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 5 }
}
```

**How to show:**
- **Table/cards:** Loop over `data.items`. For each item show: **title**, **status** (badge: DRAFT / PUBLISHED / CANCELLED), **startAt** (format as date/time, e.g. “03 Mar 2026 • TBD”), **venueName** or **city**, **tickets_sold** and **capacityTotal** (e.g. “2/100” or “2/-” if no capacity), **coverImageUrl** (prepend backend URL for `<img src>`).
- **Pagination:** Use `data.pagination.page`, `data.pagination.pageSize`, `data.pagination.total` for “Page 1 of 3” and next/prev.

---

### GET /api/v1/events/my (list with search, sort, filter)

**Query:** status, search or q, sort (soonest | oldest | title_asc | title_desc | tickets_sold), page, limit.

**Response (data):**
```json
{
  "events": [
    {
      "event_id": "27",
      "title": "My First Event",
      "status": "published",
      "event_date": "2026-03-03T11:19:00.000Z",
      "tickets_sold": 2,
      "gross_revenue_in_escrow": 11000,
      "payout_status": "pending"
    }
  ],
  "total": 5
}
```

**How to show:**
- **Dashboard “Live events” table:** Each row = one object in `data.events`. Show: **title**, **event_date** (e.g. “03 Mar 2026 • TBD”), **status** (badge: DRAFT / PUBLISHED / CANCELLED), **tickets_sold** (e.g. “2/-” or “2/100” if you have capacity from another call), **gross_revenue_in_escrow** (pence → e.g. “£110.00”), **payout_status** (e.g. “Pending”). Use **event_id** for links (e.g. “View” → `/events/27`).
- **Pagination:** Use `data.total` and your `limit` to show total count and page controls.

---

### GET /api/promoters/events/:eventId (single event detail for promoter)

**Response (data):** One event object with **all DB columns** (id, title, description, start_at, end_at, timezone, format, access_mode, visibility_mode, city, venue_name, venue_address, lat, lng, category_id, status, cover_image_url, gallery_image_urls, tickets_sold, etc.) plus:

- **promoter_name**, **category_name**, **territory_name**
- **tags:** array of `{ id, name, slug }`
- **ticketTypes:** array of ticket-type objects (see below)

**Ticket type object (inside event.ticketTypes):**
```json
{
  "id": 1,
  "name": "General",
  "description": null,
  "currency": "GBP",
  "priceAmount": 1050,
  "bookingFeeAmount": 235,
  "totalAmount": 1285,
  "salesStartAt": null,
  "salesEndAt": null,
  "capacityTotal": 100,
  "capacitySold": 2,
  "capacityRemaining": 98,
  "perOrderLimit": 10,
  "visibility": "public",
  "status": "active",
  "sortOrder": 0
}
```

**How to show:**
- **Detail/edit page:** Show title, description, dates (start_at, end_at), venue (venue_name, venue_address), city, format, access_mode, status. Show **cover_image_url** as main image (prepend backend URL); **gallery_image_urls** as a gallery strip or grid.
- **Category:** Show **category_name** (or resolve categoryId to name from categories API).
- **Tags:** Show **tags[].name** as chips/badges.
- **Ticket tiers table:** Loop **ticketTypes**. For each row show: **name**, **priceAmount/100** as “£10.50”, **bookingFeeAmount/100**, **totalAmount/100**, **capacitySold** / **capacityTotal** (e.g. “2 / 100”), **capacityRemaining**, **status** (active/paused/sold_out). Use **id** for “Edit tier” or “Pause/Resume”.

---

### GET /api/v1/events/:id (event detail for public or owner)

**Response (data):**
```json
{
  "event_id": "27",
  "title": "My First Event",
  "description": "...",
  "event_date": "2026-03-03T11:19:00.000Z",
  "venue": "The Venue",
  "status": "published",
  "promoter_id": 60,
  "escrow_protected": true,
  "ticket_tiers": [
    {
      "tier_id": "1",
      "tier_name": "General",
      "ticket_price": 10.5,
      "booking_fee": 2.35,
      "total_cost": 13.85,
      "quantity_available": 98,
      "quantity_sold": 2
    }
  ]
}
```

**How to show:**
- **Public event page / checkout:** Show **title**, **description**, **event_date**, **venue**, **status**. For tickets, loop **ticket_tiers**: show **tier_name**, **ticket_price** (“£10.50”), **booking_fee** (“+ £2.35”), **total_cost** (“£13.85”), **quantity_available** (“98 left”). Use **tier_id** as the value when calling purchase API.

---

### GET /api/promoters/events/:eventId/ticket-types (list ticket types)

**Query:** status (optional), page, pageSize.

**Response (data):**
```json
{
  "items": [
    {
      "id": 1,
      "name": "General",
      "description": null,
      "currency": "GBP",
      "priceAmount": 1050,
      "bookingFeeAmount": 235,
      "totalAmount": 1285,
      "salesStartAt": null,
      "salesEndAt": null,
      "capacityTotal": 100,
      "capacitySold": 2,
      "capacityRemaining": 98,
      "perOrderLimit": 10,
      "visibility": "public",
      "status": "active",
      "sortOrder": 0,
      "createdAt": "...",
      "updatedAt": "..."
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 3, "totalPages": 1 }
}
```

**How to show:**
- **Tiers table on event edit page:** Same as **ticketTypes** in event detail: **name**, **priceAmount/100** (“£10.50”), **bookingFeeAmount/100**, **totalAmount/100**, **capacitySold/capacityTotal**, **capacityRemaining**, **status**. Use **id** for edit/delete/pause/duplicate.

---

### GET /api/v1/events/my/export (download CSV)

**Response:** CSV file (not JSON). Header row:  
`event_id,title,status,event_date,tickets_sold,revenue_gbp,payout_status`  
Then one row per event. Response header **X-Export-Rows** = number of rows.

**How to show:** Trigger download in browser (e.g. open URL with same query params or fetch and create blob link). No UI to “display” except “Download started” or file save dialog.

---

### Image URLs in responses

- **coverImageUrl**, **galleryImageUrls**, **cover_image_url**, **imageUrl** from upload are **paths** like `/uploads/photo-123.jpg`.
- **Full URL for &lt;img src&gt;:** `BACKEND_URL + path`, e.g. `https://api.example.com/uploads/photo-123.jpg`.

---

## 6. Summary

- **Create:** Either **POST /api/promoters/events** (no tiers in body) or **POST /api/v1/events** (with ticket_tiers in one call).  
- **List/dashboard:** **GET /api/v1/events/my** (search, sort, filter, pagination) and **GET /api/v1/events/my/export** (CSV).  
- **Detail:** **GET /api/promoters/events/:eventId** (owner) or **GET /api/v1/events/:id** (public/draft-owner).  
- **Update/delete:** PATCH and DELETE on **/api/promoters/events/:eventId**.  
- **Images:** POST/DELETE/PATCH on **/api/promoters/events/:eventId/images** and **cover** / **gallery/:imageId**.  
- **Category/tags:** PUT **/api/promoters/events/:eventId/category** and **.../tags**.  
- **Go live:** **POST /api/promoters/events/:eventId/publish** (after ticket types and required fields are set).  
- **Lifecycle:** pause, cancel, republish, complete via POST to the corresponding paths.  
- **Tickets:** All ticket-type CRUD under **/api/promoters/events/:eventId/ticket-types** and **/api/promoters/ticket-types/:ticketTypeId**.  
- **On-the-day:** attendees, validate, checkin, undo checkin, logs, access settings under **/api/promoters/events/:eventId/...**.

All of the above require the user to be logged in as a promoter; event-scoped routes also require the event to belong to that promoter (enforced by requireEventOwnership / requireTicketTypeOwnership where applicable).
