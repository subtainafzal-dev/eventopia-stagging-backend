# Day 7 — Frontend API Integration (v1 Events & Tickets)

Base URL: `/api/v1`.  
Auth: `Authorization: Bearer <access_token>` for protected routes.  
Standard responses: success `{ "error": false, "data": { ... } }`, error `{ "error": true, "code": "CODE", "message": "..." }`.

---

## 1. Events

### 1.1 Create event (Promoter)

**POST** `/api/v1/events`

**Auth:** Required — **Promoter** role.

**Body (JSON):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | Yes | Event title |
| `event_date` | string | Yes | ISO8601 date/time (e.g. `2025-06-15T19:00:00.000Z`) |
| `description` | string | No | Event description |
| `venue` | string | No | Venue name or short label (also used as city_display fallback) |
| `ticket_tiers` | array | Yes | At least one tier; see below |
| `age_restriction` | any | No | Optional |
| `event_type` | any | No | Optional |

**Each element of `ticket_tiers`:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tier_name` | string | No | Display name (defaults to "Tier 1", "Tier 2", …) |
| `ticket_price` | number | Yes | Price in **GBP** (e.g. `10.50`). Must fall in tier bands (e.g. £0.01–£5.99, £6–£15.99, …). |
| `quantity_available` | number | Yes | Capacity for this tier |

**What to pass from frontend:**

- Send `ticket_tiers` with `tier_name`, `ticket_price` in **pounds** (not pence), and `quantity_available`.
- Do **not** send `booking_fee` — it is calculated by the backend (Tier Resolver).

**Success (201):**

```json
{
  "error": false,
  "data": {
    "event_id": "123",
    "status": "draft",
    "title": "My Event",
    "created_at": "2025-02-06T12:00:00.000Z"
  }
}
```

**Error codes:** `MISSING_REQUIRED_FIELDS`, `INVALID_TICKET_PRICE`, `INVALID_DATE`, `INTERNAL_ERROR`.

---

### 1.2 Event detail (public / draft owner)

**GET** `/api/v1/events/:id`

**Auth:** Optional. If sent, used to allow **draft** events to be visible only to the **owner** (promoter). For **published** events, no auth needed.

**What to pass:**

- Path: `id` = event ID (e.g. `GET /api/v1/events/42`).
- For draft events: send `Authorization: Bearer <token>` when the user is the promoter so they can see their draft.

**Success (200):**

```json
{
  "error": false,
  "data": {
    "event_id": "42",
    "title": "Event Title",
    "description": "...",
    "event_date": "2025-06-15T19:00:00.000Z",
    "venue": "Venue Name",
    "status": "draft",
    "promoter_id": 1,
    "escrow_protected": true,
    "ticket_tiers": [
      {
        "tier_id": "1",
        "tier_name": "General",
        "ticket_price": 10.5,
        "booking_fee": 2.35,
        "total_cost": 12.85,
        "quantity_available": 100,
        "quantity_sold": 0
      }
    ]
  }
}
```

**Error codes:** `EVENT_NOT_FOUND`, `DRAFT_NOT_YOUR_EVENT` (403 when draft and not owner), `INTERNAL_ERROR`.

---

### 1.3 My events (Promoter) — list + search, sort, filter

**GET** `/api/v1/events/my`

**Auth:** Required — **Promoter** role.

**Query:**

| Param | Type | Description |
|-------|------|-------------|
| `status` | string | Optional: `draft`, `published`, `live` (treated as published), `completed`, `cancelled` (for “ALL STATUS” / filter dropdown) |
| `search` or `q` | string | Optional: search events by title (e.g. “Search events…”) |
| `sort` | string | Optional: `soonest` (default), `oldest`, `title_asc`, `title_desc`, `tickets_sold` (for “SORT: SOONEST” dropdown) |
| `page` | number | Default 1 |
| `limit` | number | Default 20, max 100 |

**What to pass:**

- **Filter by status:** `status=draft` \| `published` \| `completed` \| `cancelled` (or `live` → published).
- **Search:** `search=...` or `q=...` (filters by event title).
- **Sort:** `sort=soonest` \| `oldest` \| `title_asc` \| `title_desc` \| `tickets_sold`.
- Pagination: `page` and `limit`.

**Success (200):**

```json
{
  "error": false,
  "data": {
    "events": [
      {
        "event_id": "42",
        "title": "My Event",
        "status": "published",
        "event_date": "2025-06-15T19:00:00.000Z",
        "tickets_sold": 10,
        "gross_revenue_in_escrow": 5000,
        "payout_status": "pending"
      }
    ],
    "total": 1
  }
}
```

**Note:** `gross_revenue_in_escrow` is the **territory** escrow balance (pence), not per-event. `payout_status` is fixed as `"pending"` for Day 7.

**Error codes:** `INTERNAL_ERROR`.

---

### 1.4 My events export / download (Promoter)

**GET** `/api/v1/events/my/export`

**Auth:** Required — **Promoter** role.

**Query:** Same as list (no pagination): `status`, `search` or `q`, `sort` (optional).

**Response:** CSV file download with header row: `event_id,title,status,event_date,tickets_sold,revenue_gbp,payout_status`. Response header `X-Export-Rows` = number of rows.

**What to pass:** Use the same `status`, `search`/`q`, and `sort` as on the dashboard so the download matches the current view. Trigger from the download icon; browser will save as `my_events_export.csv`.

**Error codes:** `INTERNAL_ERROR`.

---

## 2. Tickets (Buyer)

### 2.1 Purchase tickets (no payment)

**POST** `/api/v1/tickets/purchase`

**Auth:** Required — **Buyer** role.

**Body (JSON):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | number/string | Yes | Event ID |
| `ticket_tier` | number/string | Yes | **Ticket type ID** (same as `tier_id` from GET event detail) |
| `quantity` | number | Yes | Number of tickets (≥ 1) |
| `attendee_names` | string[] | Yes | Array of **exactly `quantity`** names (one per ticket) |

**What to pass from frontend:**

1. Get event and tiers from **GET /api/v1/events/:id**; use `ticket_tiers[].tier_id` as `ticket_tier`.
2. Send `event_id`, `ticket_tier` (tier id), `quantity`, and `attendee_names` with length **equal to quantity**.
3. Do **not** send payment details — purchase is confirmed immediately (no payment provider).

**Success (201):**

```json
{
  "error": false,
  "data": {
    "confirmation_reference": "EVT-1738843200000-1234",
    "ticket_ids": ["1", "2"],
    "total_paid": 25.7,
    "booking_fee_total": 4.7,
    "tickets": [
      {
        "ticket_id": "1",
        "attendee_name": "Alice",
        "tier_name": "General",
        "ticket_price": 10.5,
        "booking_fee": 2.35
      },
      {
        "ticket_id": "2",
        "attendee_name": "Bob",
        "tier_name": "General",
        "ticket_price": 10.5,
        "booking_fee": 2.35
      }
    ]
  }
}
```

**Error codes:** `MISSING_REQUIRED_FIELDS`, `ATTENDEE_NAMES_MISMATCH`, `VALIDATION_ERROR`, `EVENT_NOT_FOUND`, `EVENT_NOT_LIVE` (event not published), `TICKET_TYPE_NOT_FOUND`, `INSUFFICIENT_QUANTITY`, `INTERNAL_ERROR`.

---

### 2.2 My tickets

**GET** `/api/v1/tickets/my`

**Auth:** Required — **Buyer** role.

**Query:**

| Param | Type | Description |
|-------|------|-------------|
| `status` | string | Optional: `active`, `used`, `cancelled`, `refunded` |
| `page` | number | Default 1 |
| `limit` | number | Default 20, max 100 |

**What to pass:**

- Use `status` to filter (e.g. `active` for current tickets).
- Pagination: `page`, `limit`.

**Success (200):**

```json
{
  "error": false,
  "data": {
    "tickets": [
      {
        "ticket_id": "1",
        "event_id": 42,
        "event_title": "My Event",
        "event_date": "2025-06-15T19:00:00.000Z",
        "venue": "Venue Name",
        "tier_name": "General",
        "ticket_price": 10.5,
        "booking_fee": 2.35,
        "attendee_name": "Alice",
        "status": "ACTIVE",
        "confirmation_reference": "EVT-1738843200000-1234",
        "purchased_at": "2025-02-06T12:00:00.000Z"
      }
    ],
    "total": 1
  }
}
```

**Error codes:** `INTERNAL_ERROR`.

---

## 3. Event images — how they are handled

Event images are **not** sent in the v1 create-event body. They use a separate **upload-after-create** flow.

### Process (overview)

1. **Create event** (e.g. `POST /api/v1/events`) — event is created with no images.
2. **Upload images** using the **promoter event image APIs** (see below): one request per file (cover or gallery).
3. **Storage:** Files are saved on the server in the `uploads/` folder; the API returns a **URL path** (e.g. `/uploads/photo-1234567890.jpg`).
4. **Serving:** The backend serves files at `GET /uploads/<filename>`, so the full image URL is **`<BACKEND_URL>/uploads/<filename>`**.
5. **Data:** The app stores:
   - **Cover:** `events.cover_image_url` and one row in `event_media` with `is_cover = true`.
   - **Gallery:** Up to 10 images in `event_media` (and `events.gallery_image_urls[]` for legacy).

### APIs to use (promoter, auth required)

| Action | Method | Endpoint | Body / query |
|--------|--------|----------|--------------|
| **Upload cover image** | POST | `/api/promoter/events/:eventId/images?type=cover` | `multipart/form-data`; field name: **`image`** (file). |
| **Upload gallery image** | POST | `/api/promoter/events/:eventId/images?type=gallery` | Same; max **10** gallery images per event. |
| **Delete cover** | DELETE | `/api/promoter/events/:eventId/cover` | — |
| **Delete gallery image** | DELETE | `/api/promoter/events/:eventId/gallery/:imageId` | `imageId` = `event_media.id`. |
| **Reorder gallery** | PATCH | `/api/promoter/events/:eventId/images/reorder` | Body: `{ "imageOrder": [id1, id2, ...] }`. |

- **Upload response (cover):** `{ "mediaId", "coverImageUrl": "/uploads/..." }`.
- **Upload response (gallery):** `{ "mediaId", "imageUrl": "/uploads/...", "sortOrder" }`.
- **Full image URL in frontend:** prepend your backend base URL, e.g. `https://api.example.com/uploads/photo-123.jpg`.

### Frontend flow

1. Create event → get `event_id`.
2. For **cover:** `POST /api/promoter/events/<event_id>/images?type=cover` with form-data field **`image`** (file). Use returned `coverImageUrl` (or `BACKEND_URL + coverImageUrl`) to display.
3. For **gallery:** repeat `POST /api/promoter/events/<event_id>/images?type=gallery` with field **`image`** for each file (up to 10). Use returned `imageUrl` for display.
4. To **delete:** call the DELETE endpoints above. To **reorder** gallery, call PATCH with the desired `imageOrder` (array of `event_media.id`).

**Note:** Event **detail** APIs (e.g. `GET /api/v1/events/:id` or the public events API) return `cover_image_url` and `gallery_image_urls` so the frontend can show the images without calling the upload API again.

---

## 4. Flow summary for frontend

1. **Promoter – create event**  
   `POST /api/v1/events` with `title`, `event_date`, `ticket_tiers` (each: `tier_name`, `ticket_price` in GBP, `quantity_available`). Event is created as **draft**.

2. **Publish event**  
   Use existing app flow or API to set event `status` to **published** (e.g. PATCH event or admin). Purchase is only allowed when status is **published**.

3. **Buyer – event detail**  
   `GET /api/v1/events/:id` to show event and `ticket_tiers` (use `tier_id` for purchase).

4. **Buyer – purchase**  
   `POST /api/v1/tickets/purchase` with `event_id`, `ticket_tier` (= `tier_id` from event detail), `quantity`, `attendee_names` (length = quantity). No payment step; response includes `confirmation_reference` and `tickets`.

5. **Buyer – my tickets**  
   `GET /api/v1/tickets/my` (optional `?status=active&page=1&limit=20`) to list tickets with `confirmation_reference`, `purchased_at`, and event/venue/tier details.

6. **Promoter – my events (dashboard)**  
   `GET /api/v1/events/my` with optional `status`, `search`/`q`, `sort`, `page`, `limit` for the live events list, filters, and sort.  
   **Download:** `GET /api/v1/events/my/export?status=...&search=...&sort=...` for CSV (same filters as list).
