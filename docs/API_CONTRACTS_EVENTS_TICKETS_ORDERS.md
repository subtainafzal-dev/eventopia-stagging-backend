# API Contracts — Events, Ticket Types, and Orders

API contract document for Eventopia: **Promoter** (event CRUD, ticket types) and **Buyer** (browse events, orders, tickets). Each contract follows: **Role** → **User story** → **Table** (Module, Endpoint, Method, Who Calls It, Purpose, Auth Required, Request Body, Response Body, Error Codes, Business Rules, Calls Internally, Status).

Base URL: `/api`.

---

## Role: Buyer

**User story:** As a buyer, I can discover published events, view event details and ticket types, create orders, view my orders and tickets, and get QR codes for entry.

---

### Contract 1: List Public Events (Buyer)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `GET /api/events` |
| **Method** | GET |
| **Who Calls It** | Any user (authenticated or not) via Event Discovery / Browse Screen |
| **Purpose** | Returns a paginated list of published, public events with optional filters and sort. |
| **Auth Required** | No |
| **Request Body** | N/A (query params only) |
| **Query Params** | `city: string` (optional), `categoryId: number` (optional), `tagIds: number[]` or comma-separated (optional), `dateFrom: ISO8601` (optional), `dateTo: ISO8601` (optional), `search: string` (optional), `sort: enum [soonest \| newest \| popular]` (default soonest), `page: number` (default 1), `pageSize: number` (default 20, max 50) |
| **Response Body** | `{ items: Array<{ id, title, startAt, endAt, city, venueName, format, accessMode, category: { id, name, slug }, tags: Array<{ id, name, slug }>, coverImageUrl, ticketsSold }>, pagination: { page, pageSize, total } }` |
| **Error Codes** | `500 INTERNAL_ERROR` |
| **Business Rules** | Only events with `status = 'published'` and `visibility_mode = 'public'` are returned. |
| **Calls Internally** | DB: events, categories, event_tags, tags |
| **Status** | IMPLEMENTED |

---

### Contract 2: Get Event Detail (Buyer — Public)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `GET /api/events/:id` |
| **Method** | GET |
| **Who Calls It** | Any user via Event Detail Screen (public event page) |
| **Purpose** | Returns full event details and public ticket types for a published public event. |
| **Auth Required** | No |
| **Request Body** | N/A |
| **Response Body** | `{ event: { id, title, description, startAt, endAt, timezone, city, venueName, venueAddress, format, accessMode, category, tags, promoter: { name }, images: { coverImageUrl, galleryImageUrls } }, ticketTypes: Array<{ id, name, description, currency, priceAmount, bookingFeeAmount, totalAmount, salesStartAt, salesEndAt, capacityTotal, capacitySold, capacityRemaining, perOrderLimit, status }> }` |
| **Error Codes** | `404 NOT_FOUND` (event not found or not accessible) \| `500 INTERNAL_ERROR` |
| **Business Rules** | Event must be `status = 'published'` and `visibility_mode = 'public'`. Ticket types returned only when `access_mode` is ticketed or mixed; only `visibility = 'public'` and `status != 'hidden'`. |
| **Calls Internally** | DB: events, categories, event_tags, tags, ticket_types |
| **Status** | IMPLEMENTED |

---

### Contract 3: Get Event by Share Token (Buyer — Private Link)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `GET /api/events/share/:shareToken` |
| **Method** | GET |
| **Who Calls It** | User with private event link via Private Link Event Page |
| **Purpose** | Returns event details for a private-link event and sets an access-grant cookie so the user can create orders. |
| **Auth Required** | No (optional auth; cookie set for ordering) |
| **Request Body** | N/A |
| **Response Body** | Same shape as Contract 2 (event + ticketTypes). Response also sets `event_access_grant` cookie. |
| **Error Codes** | `404 NOT_FOUND` \| `500 INTERNAL_ERROR` |
| **Business Rules** | Event must have `share_token` matching param, `status = 'published'`, `visibility_mode = 'private_link'`. Access grant cookie required for creating orders on private/hidden events. |
| **Calls Internally** | access.service (generateAccessGrant), DB: events, categories, tags, ticket_types |
| **Status** | IMPLEMENTED |

---

### Contract 4: Create Order (Buyer)

| Field | Value |
|-------|--------|
| **Module** | Orders |
| **Endpoint** | `POST /api/orders/events/:eventId/orders` |
| **Method** | POST |
| **Who Calls It** | Authenticated buyer via Checkout Screen |
| **Purpose** | Creates a new order (payment_pending) with reserved inventory for the given event and ticket type quantities. |
| **Auth Required** | Yes (Bearer) |
| **Request Body** | `{ idempotencyKey: string (required), items: Array<{ ticketTypeId: number (required), quantity: number (required, min 1) }> (required), buyerName: string (optional), buyerEmail: string (optional) }` |
| **Response Body** | `{ orderId: number, status: string, totalAmount: number, currency: string, expiresAt: ISO8601 }` |
| **Error Codes** | `400 VALIDATION_ERROR` (missing idempotencyKey, invalid items, invalid eventId) \| `401 UNAUTHORIZED` \| `403 ACCESS_DENIED` (private/hidden event without access grant) \| `404 EVENT_NOT_FOUND` \| `404 TICKET_TYPE_NOT_FOUND` \| `400 EVENT_NOT_PUBLISHED` \| `400 TICKET_TYPE_NOT_ACTIVE` \| `500 INTERNAL_ERROR` |
| **Business Rules** | Event must be published. For private_link or hidden ticket types, valid `event_access_grant` cookie required. All ticket type IDs must belong to the event; all ticket types must be active. Inventory reserved via `inventory_reservations`; order expires after 30 minutes if not paid. |
| **Calls Internally** | access.service (hasEventAccess), DB: events, ticket_types, orders, order_items, inventory_reservations |
| **Status** | IMPLEMENTED |

---

### Contract 5: Get Order Details (Buyer)

| Field | Value |
|-------|--------|
| **Module** | Orders |
| **Endpoint** | `GET /api/orders/:orderId` |
| **Method** | GET |
| **Who Calls It** | Authenticated buyer via Order Confirmation / Order Detail Screen |
| **Purpose** | Returns full order details including event snapshot and line items (ticket types and ticket IDs). |
| **Auth Required** | Yes (Bearer) |
| **Request Body** | N/A |
| **Response Body** | `{ order: { id, orderNumber, status, totalAmount, currency, createdAt, eventSnapshot: { title, startAt, format, venueName, cityDisplay, coverImageUrl }, items: Array<{ itemId, quantity, subtotalAmount, ticketTypeName, ticketId, ticketStatus, usedAt }> } }` |
| **Error Codes** | `404 ORDER_NOT_FOUND` \| `403 FORBIDDEN` (not order owner) \| `500 INTERNAL_ERROR` |
| **Business Rules** | Only the buyer who created the order can view it. |
| **Calls Internally** | DB: orders, events, order_items, ticket_types, tickets |
| **Status** | IMPLEMENTED |

---

### Contract 6: Get My Orders (Buyer)

| Field | Value |
|-------|--------|
| **Module** | Orders |
| **Endpoint** | `GET /api/orders/me/orders` |
| **Method** | GET |
| **Who Calls It** | Authenticated buyer via My Orders Screen |
| **Purpose** | Returns paginated list of the current user's orders. |
| **Auth Required** | Yes (Bearer) |
| **Request Body** | N/A |
| **Query Params** | `page: number` (default 1), `pageSize: number` (default 20, max 50), `status: string` (optional), `upcoming: boolean` (optional, true = event start >= NOW()) |
| **Response Body** | `{ orders: Array<{ orderId, orderNumber, totalAmount, status, createdAt, eventSnapshot: { ... }, ticketsCount, activeTicketsCount }>, pagination: { page, pageSize, total } }` |
| **Error Codes** | `500 INTERNAL_ERROR` |
| **Business Rules** | Only orders where `buyer_user_id` = current user. |
| **Calls Internally** | DB: orders, events, order_items, tickets |
| **Status** | IMPLEMENTED |

---

### Contract 7: Get My Tickets (Buyer)

| Field | Value |
|-------|--------|
| **Module** | Orders / Tickets |
| **Endpoint** | `GET /api/orders/me/tickets` |
| **Method** | GET |
| **Who Calls It** | Authenticated buyer via My Tickets Screen |
| **Purpose** | Returns paginated list of the current user's tickets with event info. |
| **Auth Required** | Yes (Bearer) |
| **Request Body** | N/A |
| **Query Params** | `page: number`, `pageSize: number`, `status: string` (optional), `from: ISO8601` (optional), `to: ISO8601` (optional) |
| **Response Body** | `{ tickets: Array<{ ticketId, status, ticketCode, issuedAt, usedAt, refundedAt, cancelledAt, ticketTypeName, event: { id, title, startAt, format, venueName, cityDisplay, coverImageUrl } }>, pagination: { page, pageSize, total } }` |
| **Error Codes** | `500 INTERNAL_ERROR` |
| **Business Rules** | Only tickets belonging to the current user. |
| **Calls Internally** | DB: tickets, ticket_types, events |
| **Status** | IMPLEMENTED |

---

### Contract 8: Get Order Tickets (Buyer)

| Field | Value |
|-------|--------|
| **Module** | Orders |
| **Endpoint** | `GET /api/orders/:orderId/tickets` |
| **Method** | GET |
| **Who Calls It** | Authenticated buyer via Order Detail Screen |
| **Purpose** | Returns tickets belonging to a specific order. |
| **Auth Required** | Yes (Bearer) |
| **Request Body** | N/A |
| **Response Body** | Array of ticket objects for the order (structure as per implementation). |
| **Error Codes** | `404 ORDER_NOT_FOUND` \| `403 FORBIDDEN` \| `500 INTERNAL_ERROR` |
| **Business Rules** | Only order owner can list tickets. |
| **Calls Internally** | DB: orders, order_items, tickets |
| **Status** | IMPLEMENTED |

---

### Contract 9: Get Ticket Details (Buyer)

| Field | Value |
|-------|--------|
| **Module** | Orders / Tickets |
| **Endpoint** | `GET /api/orders/tickets/:ticketId` |
| **Method** | GET |
| **Who Calls It** | Authenticated buyer via Ticket Detail Screen |
| **Purpose** | Returns details for a single ticket. |
| **Auth Required** | Yes (Bearer) |
| **Request Body** | N/A |
| **Response Body** | Ticket object (id, status, ticketCode, event snapshot, etc. as implemented). |
| **Error Codes** | `404 INVALID_TICKET_ID` \| `403 FORBIDDEN` \| `500 INTERNAL_ERROR` |
| **Business Rules** | Only the buyer who owns the ticket can view it. |
| **Calls Internally** | DB: tickets, order_items, orders |
| **Status** | IMPLEMENTED |

---

### Contract 10: Get Ticket QR (Buyer)

| Field | Value |
|-------|--------|
| **Module** | Orders / Tickets |
| **Endpoint** | `GET /api/orders/tickets/:ticketId/qr` |
| **Method** | GET |
| **Who Calls It** | Authenticated buyer via Ticket / Entry Screen (to show QR for scanning) |
| **Purpose** | Returns QR payload for a ticket so the buyer can display it at entry. |
| **Auth Required** | Yes (Bearer) |
| **Request Body** | N/A |
| **Response Body** | `{ ticketId: number, qrPayload: string, status: string, expiresIn: number }` |
| **Error Codes** | `404 TICKET_NOT_FOUND` \| `403 FORBIDDEN` \| `400 INVALID_STATUS` (QR not available for non-ACTIVE tickets) \| `500 INTERNAL_ERROR` |
| **Business Rules** | Only ACTIVE tickets return a QR payload. Ownership verified via order buyer. |
| **Calls Internally** | qr.service (generateTicketQR), DB: tickets, order_items, orders |
| **Status** | IMPLEMENTED |

---

### Contract 11: Cancel Order (Buyer)

| Field | Value |
|-------|--------|
| **Module** | Orders |
| **Endpoint** | `POST /api/orders/:orderId/cancel` |
| **Method** | POST |
| **Who Calls It** | Authenticated buyer via Order Detail / My Orders Screen |
| **Purpose** | Cancels an order and releases inventory reservation. |
| **Auth Required** | Yes (Bearer) |
| **Request Body** | Optional body (e.g. reason) as per implementation. |
| **Response Body** | `{ orderId, status: 'cancelled' }` or similar. |
| **Error Codes** | `404 ORDER_NOT_FOUND` \| `403 FORBIDDEN` \| `400 INVALID_STATE` (e.g. already completed/cancelled) \| `500 INTERNAL_ERROR` |
| **Business Rules** | Only order owner can cancel. Only orders in cancellable state (e.g. PENDING/payment_pending). |
| **Calls Internally** | DB: orders, inventory_reservations |
| **Status** | IMPLEMENTED |

---

### Contract 12: Checkout Order (Buyer)

| Field | Value |
|-------|--------|
| **Module** | Orders |
| **Endpoint** | `POST /api/orders/:orderId/checkout` |
| **Method** | POST |
| **Who Calls It** | Authenticated buyer via Checkout Flow (after payment) |
| **Purpose** | Completes the order (e.g. confirm payment, convert reservations to tickets). |
| **Auth Required** | Yes (Bearer) |
| **Request Body** | As per implementation (e.g. payment intent id). |
| **Response Body** | Order confirmation (status, tickets, etc. as implemented). |
| **Error Codes** | `404 ORDER_NOT_FOUND` \| `403 FORBIDDEN` \| `400 INVALID_STATE` \| `500 INTERNAL_ERROR` |
| **Business Rules** | Only order owner. Order must be in payment_pending/PENDING state; reservations must not be expired. |
| **Calls Internally** | DB: orders, order_items, inventory_reservations, tickets; payment service when integrated |
| **Status** | IMPLEMENTED |

---

## Role: Promoter

**User story:** As a promoter, I can create and manage my events (draft, publish, pause, cancel), manage ticket types, upload images, set category and tags, and view event performance.

---

### Contract 13: Create Event (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `POST /api/promoters/events` |
| **Method** | POST |
| **Who Calls It** | Authenticated promoter with active account via Create Event Screen |
| **Purpose** | Creates a new draft event with derived hierarchy (guru, network manager, territory). |
| **Auth Required** | Yes (Bearer); promoter must be active |
| **Request Body** | `{ title: string (optional), description: string (optional), startAt: ISO8601 (optional), endAt: ISO8601 (optional), timezone: string (default "Europe/London"), format: enum [in_person \| online_live \| virtual_on_demand \| hybrid] (default "in_person"), accessMode: enum [ticketed \| guest_list \| mixed] (default "ticketed"), visibilityMode: enum [public \| private_link] (default "public"), city: string (optional), venueName: string (optional), venueAddress: string (optional), lat: number (optional), lng: number (optional), categoryId: number (optional), tagIds: number[] (optional), tagNames: string[] (optional) }` — at least one of title, description, city, startAt, endAt required |
| **Response Body** | `{ id: number, message: "Event created successfully" }` |
| **Error Codes** | `400 VALIDATION_FAILED` (no fields provided) \| `403 FORBIDDEN` (active promoter required) \| `500 INTERNAL_ERROR` |
| **Business Rules** | Promoter must have `account_status = 'active'`. Hierarchy (guru_id, network_manager_id, territory_id) derived from promoter. If `visibilityMode = 'private_link'`, a share_token is generated. |
| **Calls Internally** | deriveHierarchyFromPromoter, DB: events, event_tags, tags |
| **Status** | IMPLEMENTED |

---

### Contract 14: List Promoter Events (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `GET /ap` |
| **Method** | GET |
| **Who Calls It** | Authenticated promoter via Dashboard / My Events Screen |
| **Purpose** | Returns paginated list of the promoter's events with optional status, sort, search. |
| **Auth Required** | Yes (Bearer); active promoter |
| **Request Body** | N/A |
| **Query Params** | `status: string` (optional), `sort: string` (default "updated"), `page: number`, `pageSize: number`, `search: string` (optional) |
| **Response Body** | `{ items: Array<event row with id, title, status, start_at, etc.>, pagination: { page, pageSize, total } }` |
| **Error Codes** | `500 INTERNAL_ERROR` |
| **Business Rules** | Only events where `promoter_id` = current user. |
| **Calls Internally** | DB: events |
| **Status** | IMPLEMENTED |

---

### Contract 15: Get Promoter Event Detail (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `GET /api/promoters/events/:eventId` |
| **Method** | GET |
| **Who Calls It** | Authenticated promoter via Edit Event / Event Detail Screen |
| **Purpose** | Returns full event detail including tags and ticket types with availability (for owner only). |
| **Auth Required** | Yes (Bearer); active promoter; event ownership required |
| **Request Body** | N/A |
| **Response Body** | Full event object with `tags[]`, `ticketTypes[]` (id, name, priceAmount, bookingFeeAmount, totalAmount, salesStartAt, salesEndAt, capacityTotal, capacitySold, capacityRemaining, perOrderLimit, visibility, status, sortOrder). |
| **Error Codes** | `404 NOT_FOUND` (event not found or not owner) \| `500 INTERNAL_ERROR` |
| **Business Rules** | Event must belong to current promoter. |
| **Calls Internally** | DB: events, users, categories, territories, event_tags, tags, ticket_types |
| **Status** | IMPLEMENTED |

---

### Contract 16: Update Event (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `PATCH /api/promoters/events/:eventId` |
| **Method** | PATCH |
| **Who Calls It** | Authenticated promoter via Edit Event Screen |
| **Purpose** | Updates event fields (partial update). Cancelled events cannot be edited. |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | Any subset of: `title, description, startAt, endAt, timezone, format, accessMode, visibilityMode, city, venueName, venueAddress, lat, lng, categoryId, tagIds, tagNames, resetShareToken: boolean` |
| **Response Body** | `{ id: number }` |
| **Error Codes** | `404 NOT_FOUND` \| `400 INVALID_STATE` (cancelled event) \| `500 INTERNAL_ERROR` |
| **Business Rules** | Cannot edit cancelled events. If `resetShareToken` or switching to private_link without token, new share_token generated. Tags replaced when tagIds/tagNames provided. |
| **Calls Internally** | attachTagsToEvent, DB: events, event_tags, tags |
| **Status** | IMPLEMENTED |

---

### Contract 17: Delete Event (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `DELETE /api/promoters/events/:eventId` |
| **Method** | DELETE |
| **Who Calls It** | Authenticated promoter via Event Settings / Delete Event |
| **Purpose** | Permanently deletes an event. Allowed only when no tickets sold and event not published. |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | N/A |
| **Response Body** | `{ message: "Event deleted successfully", eventId: number }` |
| **Error Codes** | `404 NOT_FOUND` \| `403 FORBIDDEN` \| `400 TICKETS_SOLD` (cannot delete if tickets sold) \| `400 CANNOT_DELETE` (published event — cancel or pause first) \| `500 INTERNAL_ERROR` |
| **Business Rules** | Event must have `tickets_sold = 0` and must not be `status = 'published'`. |
| **Calls Internally** | DB: events (cascade deletes), audit log |
| **Status** | IMPLEMENTED |

---

### Contract 18: Upload Event Image (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `POST /api/promoters/events/:eventId/images?type=cover \| gallery` |
| **Method** | POST |
| **Who Calls It** | Authenticated promoter via Event Images Screen |
| **Purpose** | Uploads a cover or gallery image (multipart/form-data). |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | multipart/form-data, field name `image` (file) |
| **Response Body** | `{ mediaId: number, url: string, message: string }` (cover) or `{ mediaId, url, sortOrder, message }` (gallery) |
| **Error Codes** | `400 NO_FILE` \| `404 NOT_FOUND` \| `403 FORBIDDEN` \| `500 INTERNAL_ERROR` |
| **Business Rules** | For type=cover, existing cover is unset. Gallery has max limit per event (e.g. 10). |
| **Calls Internally** | upload middleware, DB: event_media, events |
| **Status** | IMPLEMENTED |

---

### Contract 19: Set Event Category (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `PUT /api/promoters/events/:eventId/category` |
| **Method** | PUT |
| **Who Calls It** | Authenticated promoter via Edit Event Screen |
| **Purpose** | Sets or clears the event's category. |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | `{ categoryId: number \| null }` |
| **Response Body** | `{ message: "Category updated successfully" }` |
| **Error Codes** | `404 NOT_FOUND` \| `500 INTERNAL_ERROR` |
| **Business Rules** | categoryId can be null to unset. |
| **Calls Internally** | DB: events |
| **Status** | IMPLEMENTED |

---

### Contract 20: Set Event Tags (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `PUT /api/promoters/events/:eventId/tags` |
| **Method** | PUT |
| **Who Calls It** | Authenticated promoter via Edit Event Screen |
| **Purpose** | Replaces event tags by IDs and/or creates tags by name. |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | `{ tagIds: number[] (optional), tagNames: string[] (optional) }` |
| **Response Body** | `{ message: "Tags updated successfully" }` |
| **Error Codes** | `500 INTERNAL_ERROR` |
| **Business Rules** | Existing event_tags for event are replaced. tagNames create new tags if not found (case-insensitive match). |
| **Calls Internally** | attachTagsToEvent, DB: event_tags, tags |
| **Status** | IMPLEMENTED |

---

### Contract 21: Publish Event (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `POST /api/promoters/events/:eventId/publish` |
| **Method** | POST |
| **Who Calls It** | Authenticated promoter via Event Publish Screen |
| **Purpose** | Sets event status to published after validating required fields and at least one active ticket type (if ticketed/mixed). |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | N/A |
| **Response Body** | `{ id: number, status: 'published', publishedAt: ISO8601 }` |
| **Error Codes** | `404 NOT_FOUND` \| `400 INVALID_STATE` (already published or cancelled) \| `400 VALIDATION_FAILED` (missing required fields or no active ticket type) \| `500 INTERNAL_ERROR` |
| **Business Rules** | Required: title, description, startAt, endAt, city, format, accessMode. If in_person/hybrid: venueName, venueAddress. If access_mode ticketed/mixed: at least one ticket type with status active. |
| **Calls Internally** | DB: events, ticket_types; audit (logEventChange) |
| **Status** | IMPLEMENTED |

---

### Contract 22: Pause Event (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `POST /api/promoters/events/:eventId/pause` |
| **Method** | POST |
| **Who Calls It** | Authenticated promoter via Event Dashboard |
| **Purpose** | Sets event status to paused (only published events). |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | N/A |
| **Response Body** | `{ id: number, status: 'paused' }` |
| **Error Codes** | `400 INVALID_STATE` (only published events can be paused) \| `404 NOT_FOUND` \| `500 INTERNAL_ERROR` |
| **Business Rules** | Current status must be published. |
| **Calls Internally** | DB: events; audit |
| **Status** | IMPLEMENTED |

---

### Contract 23: Cancel Event (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `POST /api/promoters/events/:eventId/cancel` |
| **Method** | POST |
| **Who Calls It** | Authenticated promoter via Event Dashboard |
| **Purpose** | Sets event status to cancelled and optionally stores a reason. |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | `{ reason: string (optional) }` |
| **Response Body** | `{ id: number, status: 'cancelled' }` |
| **Error Codes** | `404 NOT_FOUND` \| `400 INVALID_STATE` (already cancelled) \| `500 INTERNAL_ERROR` |
| **Business Rules** | cancel_reason stored when provided. |
| **Calls Internally** | DB: events; audit |
| **Status** | IMPLEMENTED |

---

### Contract 24: Republish Event (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `POST /api/promoters/events/:eventId/republish` |
| **Method** | POST |
| **Who Calls It** | Authenticated promoter via Event Dashboard (after pause) |
| **Purpose** | Sets event status back to published from paused. |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | N/A |
| **Response Body** | `{ id: number, status: 'published' }` |
| **Error Codes** | `400 INVALID_STATE` (only paused events can be republished) \| `404 NOT_FOUND` \| `500 INTERNAL_ERROR` |
| **Business Rules** | Current status must be paused. |
| **Calls Internally** | DB: events; audit |
| **Status** | IMPLEMENTED |

---

### Contract 25: Complete Event (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `POST /api/promoters/events/:eventId/complete` |
| **Method** | POST |
| **Who Calls It** | Authenticated promoter via Event Dashboard (post-event) |
| **Purpose** | Marks event as completed (completion_status, completed_at). |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | N/A |
| **Response Body** | `{ id: number, completionStatus: 'completed', completedAt: ISO8601 }` |
| **Error Codes** | `404 NOT_FOUND` \| `400 INVALID_STATE` (cancelled events cannot be completed) \| `500 INTERNAL_ERROR` |
| **Business Rules** | Cancelled events cannot be completed. |
| **Calls Internally** | DB: events; audit |
| **Status** | IMPLEMENTED |

---

### Contract 26: Get Event Performance (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `GET /api/promoters/events/:eventId/performance` |
| **Method** | GET |
| **Who Calls It** | Authenticated promoter via Event Analytics Screen |
| **Purpose** | Returns event metrics (tickets sold, views, conversion rate, etc.). |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | N/A |
| **Response Body** | `{ eventId: number, ticketsSold: number, viewsCount: number, conversionRate: number, grossRevenue: number, bookingFeesCollected: number, refundsTotal: number, generatedAt: ISO8601 }` |
| **Error Codes** | `404 NOT_FOUND` \| `500 INTERNAL_ERROR` |
| **Business Rules** | Owner only. |
| **Calls Internally** | DB: events |
| **Status** | IMPLEMENTED |

---

### Contract 27: Delete Event Cover / Gallery Image (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `DELETE /api/promoters/events/:eventId/cover` or `DELETE /api/promoters/events/:eventId/gallery/:imageId` |
| **Method** | DELETE |
| **Who Calls It** | Authenticated promoter via Event Images Screen |
| **Purpose** | Removes cover image or a single gallery image. |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | N/A |
| **Response Body** | `{ message: "Cover image deleted successfully" }` or `{ message: "Gallery image deleted successfully" }` |
| **Error Codes** | `404 NOT_FOUND` \| `403 FORBIDDEN` \| `400 NO_IMAGE` (cover) \| `404 IMAGE_NOT_FOUND` (gallery) \| `500 INTERNAL_ERROR` |
| **Business Rules** | imageId in gallery = event_media.id. Legacy events.cover_image_url / gallery_image_urls updated for compatibility. |
| **Calls Internally** | DB: events, event_media; audit |
| **Status** | IMPLEMENTED |

---

### Contract 28: Reorder Gallery Images (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Events |
| **Endpoint** | `PATCH /api/promoters/events/:eventId/images/reorder` |
| **Method** | PATCH |
| **Who Calls It** | Authenticated promoter via Event Images Screen |
| **Purpose** | Reorders gallery images by providing new order of media IDs. |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | `{ imageOrder: number[] }` (array of event_media.id) |
| **Response Body** | `{ message: "Gallery images reordered successfully" }` |
| **Error Codes** | `400 VALIDATION_FAILED` (imageOrder must be array) \| `404 NOT_FOUND` \| `500 INTERNAL_ERROR` |
| **Business Rules** | sort_order updated per media item; legacy gallery_image_urls array synced. |
| **Calls Internally** | DB: event_media, events; audit |
| **Status** | IMPLEMENTED |

---

## Ticket Types (Promoter)

---

### Contract 29: List Ticket Types (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Ticket Types |
| **Endpoint** | `GET /api/promoters/events/:eventId/ticket-types` |
| **Method** | GET |
| **Who Calls It** | Authenticated promoter via Event Ticket Types Screen |
| **Purpose** | Returns paginated ticket types for the event with availability. |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | N/A |
| **Query Params** | `status: string` (optional, filter by status), `page: number` (default 1), `pageSize: number` (default 20, max 50) |
| **Response Body** | `{ items: Array<{ id, name, description, currency, priceAmount, bookingFeeAmount, totalAmount, salesStartAt, salesEndAt, capacityTotal, capacitySold, capacityRemaining, perOrderLimit, visibility, status, sortOrder, createdAt, updatedAt }>, pagination: { page, pageSize, total, totalPages } }` |
| **Error Codes** | `404 INVALID_EVENT_ID` \| `500 INTERNAL_ERROR` |
| **Business Rules** | Event must belong to current promoter. |
| **Calls Internally** | DB: ticket_types |
| **Status** | IMPLEMENTED |

---

### Contract 30: Create Ticket Type (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Ticket Types |
| **Endpoint** | `POST /api/promoters/events/:eventId/ticket-types` |
| **Method** | POST |
| **Who Calls It** | Authenticated promoter via Add Ticket Type Screen |
| **Purpose** | Creates a new ticket type for the event. |
| **Auth Required** | Yes (Bearer); active promoter; event ownership |
| **Request Body** | `{ name: string (required), description: string (optional), priceAmount: number (required, non-negative integer), bookingFeeAmount: number (default 0), currency: string (default "GBP"), salesStartAt: ISO8601 (optional), salesEndAt: ISO8601 (optional), capacityTotal: number (optional), perOrderLimit: number (default 10, 1–100), visibility: enum [public \| hidden] (default "public"), status: string (default "active"), sortOrder: number (default 0), access_mode: enum [IN_PERSON \| ONLINE_LIVE \| ON_DEMAND] (default "IN_PERSON"), reveal_rule: enum [AT_PURCHASE \| ONE_HOUR_BEFORE \| AT_START] (optional, ONLINE_LIVE only), on_demand_start_at: ISO8601 (optional), on_demand_end_at: ISO8601 (optional) }` |
| **Response Body** | `{ id: number, message: "Ticket type created successfully" }` |
| **Error Codes** | `404 INVALID_EVENT_ID` \| `400 VALIDATION_ERROR` (name required, price/capacity/visibility/access_mode/reveal_rule/on_demand rules) \| `404 EVENT_NOT_FOUND` \| `500 INTERNAL_ERROR` |
| **Business Rules** | access_mode ON_DEMAND requires on_demand_start_at/on_demand_end_at; reveal_rule only for ONLINE_LIVE; on_demand_start_at < on_demand_end_at. |
| **Calls Internally** | DB: events, ticket_types |
| **Status** | IMPLEMENTED |

---

### Contract 31: Update Ticket Type (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Ticket Types |
| **Endpoint** | `PATCH /api/promoters/ticket-types/:ticketTypeId` |
| **Method** | PATCH |
| **Who Calls It** | Authenticated promoter via Edit Ticket Type Screen |
| **Purpose** | Partially updates a ticket type. Price/capacity rules apply (e.g. no price change after sales). |
| **Auth Required** | Yes (Bearer); active promoter; ticket type ownership (via event) |
| **Request Body** | Any subset of: `name, description, price_amount, booking_fee_amount, sales_start_at, sales_end_at, capacity_total, per_order_limit, visibility, status, sort_order, access_mode, reveal_rule, on_demand_start_at, on_demand_end_at` (camelCase accepted where implemented) |
| **Response Body** | `{ id: number, message: "Ticket type updated successfully" }` |
| **Error Codes** | `404 TICKET_TYPE_NOT_FOUND` \| `400 CANNOT_MODIFY_AFTER_SALE` (price) \| `400 CAPACITY_BELOW_SOLD` \| `400 VALIDATION_ERROR` (visibility, status, access_mode, etc.) \| `403 FORBIDDEN` \| `500 INTERNAL_ERROR` |
| **Business Rules** | Cannot change price or booking fee after qty_sold > 0. Capacity cannot be set below qty_sold. visibility: public|hidden; status: active|hidden|ended. |
| **Calls Internally** | DB: ticket_types, events |
| **Status** | IMPLEMENTED |

---

### Contract 32: Duplicate Ticket Type (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Ticket Types |
| **Endpoint** | `POST /api/promoters/ticket-types/:ticketTypeId/duplicate` |
| **Method** | POST |
| **Who Calls It** | Authenticated promoter via Ticket Types Screen |
| **Purpose** | Creates a copy of the ticket type (same event) with optional name, capacity, and sales window shift. |
| **Auth Required** | Yes (Bearer); active promoter; ticket type ownership |
| **Request Body** | `{ name: string (optional), capacityTotal: number (optional), adjustSalesWindow: { shiftByDays: number } (optional) }` |
| **Response Body** | `{ id: number, message: "Ticket type duplicated successfully" }` |
| **Error Codes** | `404 TICKET_TYPE_NOT_FOUND` \| `403 FORBIDDEN` \| `500 INTERNAL_ERROR` |
| **Business Rules** | New ticket type gets status active; sort_order = original + 1. Default name = "{Original name} (Copy)". |
| **Calls Internally** | DB: ticket_types, events |
| **Status** | IMPLEMENTED |

---

### Contract 33: Pause Ticket Type (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Ticket Types |
| **Endpoint** | `POST /api/promoters/ticket-types/:ticketTypeId/pause` |
| **Method** | POST |
| **Who Calls It** | Authenticated promoter via Ticket Types Screen |
| **Purpose** | Sets ticket type status to hidden (paused for sales). |
| **Auth Required** | Yes (Bearer); active promoter; ticket type ownership |
| **Request Body** | N/A |
| **Response Body** | `{ id: number, message: "Ticket type paused successfully" }` |
| **Error Codes** | `404 TICKET_TYPE_NOT_FOUND` \| `403 FORBIDDEN` \| `500 INTERNAL_ERROR` |
| **Business Rules** | status set to 'hidden'. |
| **Calls Internally** | DB: ticket_types, events |
| **Status** | IMPLEMENTED |

---

### Contract 34: Resume Ticket Type (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Ticket Types |
| **Endpoint** | `POST /api/promoters/ticket-types/:ticketTypeId/resume` |
| **Method** | POST |
| **Who Calls It** | Authenticated promoter via Ticket Types Screen |
| **Purpose** | Sets ticket type status back to active. |
| **Auth Required** | Yes (Bearer); active promoter; ticket type ownership |
| **Request Body** | N/A |
| **Response Body** | `{ id: number, message: "Ticket type resumed successfully" }` |
| **Error Codes** | `404 TICKET_TYPE_NOT_FOUND` \| `403 FORBIDDEN` \| `500 INTERNAL_ERROR` |
| **Business Rules** | status set to 'active'. |
| **Calls Internally** | DB: ticket_types, events |
| **Status** | IMPLEMENTED |

---

### Contract 35: Delete Ticket Type (Promoter)

| Field | Value |
|-------|--------|
| **Module** | Ticket Types |
| **Endpoint** | `DELETE /api/promoters/ticket-types/:ticketTypeId` |
| **Method** | DELETE |
| **Who Calls It** | Authenticated promoter via Ticket Types Screen |
| **Purpose** | Permanently deletes a ticket type. Allowed only when no tickets sold. |
| **Auth Required** | Yes (Bearer); active promoter; ticket type ownership |
| **Request Body** | N/A |
| **Response Body** | `{ message: "Ticket type deleted successfully" }` |
| **Error Codes** | `404 TICKET_TYPE_NOT_FOUND` \| `403 FORBIDDEN` \| `400 CANNOT_DELETE_AFTER_SALE` \| `500 INTERNAL_ERROR` |
| **Business Rules** | qty_sold must be 0. |
| **Calls Internally** | DB: ticket_types, events |
| **Status** | IMPLEMENTED |

---

## Summary

| Role | Contracts | Scope |
|------|-----------|--------|
| **Buyer** | 1–12 | Events (list, detail, share), Orders (create, get, list, cancel, checkout), Tickets (my tickets, order tickets, ticket detail, QR) |
| **Promoter** | 13–28 | Events CRUD, images, category, tags, publish/pause/cancel/republish/complete, performance |
| **Promoter** | 29–35 | Ticket types: list, create, update, duplicate, pause, resume, delete |

All endpoints use standard response shape: `{ data, error, message, requestId }` (see `utils/standardResponse`). Error responses include `code` and HTTP status as in the table.
