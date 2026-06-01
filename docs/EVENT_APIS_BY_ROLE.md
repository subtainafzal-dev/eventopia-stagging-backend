# Event APIs and Flow by Role

All event-related endpoints, grouped by who calls them. Base path: `/api`.

---

## Role summary

| Role | Event-related APIs |
|------|--------------------|
| **Buyer** | List public events, event detail, private-link event, create order, checkout, my orders/tickets |
| **Promoter** | CRUD events, images, publish/pause/cancel/republish/complete, ticket types, attendees, validate/check-in, logs, access settings |
| **Admin** | List/get all events, complete event, cancel event, audit logs, metrics |

**Guru** and **Network Manager** have no event-specific routes in this codebase; they operate via promoter linkage and admin/guru flows.

---

## Full example (buyer purchase flow)

### POST /api/orders/events/:eventId/orders (Orders / Ticket Purchase)

| Field | Value |
|-------|--------|
| **Module** | Orders / Credit flow |
| **Endpoint** | POST /api/orders/events/:eventId/orders |
| **Method** | POST |
| **Who calls it** | Buyer (authenticated) |
| **Purpose** | Buyer creates an order (basket) for tickets. Reserves inventory; payment completed via checkout. |
| **Auth required** | Yes. Authenticated user (buyer) |
| **Path params** | `eventId` (number, required) — event ID |
| **Request body** | `idempotencyKey` (string, required) — e.g. UUID<br>`items` (array, required) — each: `ticketTypeId` (number), `quantity` (number), optional `buyerName`, `buyerEmail` |
| **Response body** | `orderId` (number), `status` (string, e.g. "PENDING"), `totalAmount` (number, pence), `currency` ("GBP"), `expiresAt` (ISO string) |
| **Error codes** | 400 VALIDATION_ERROR — missing/invalid body<br>400 EVENT_NOT_PUBLISHED — event not published<br>403 ACCESS_DENIED — private-link or hidden tickets without access grant<br>404 EVENT_NOT_FOUND — event_id invalid<br>404 TICKET_TYPE_NOT_FOUND — ticket type not for this event<br>400 TICKET_TYPE_NOT_ACTIVE — ticket type not active<br>401 UNAUTHORIZED — not logged in |
| **Business rules** | Event must be published. For private_link or hidden ticket types, valid access grant (cookie) required. Ticket types must belong to event and be active. Inventory reserved via `inventory_reservations`; `qty_sold` updated on checkout. |
| **Calls internally** | Order insert, order_items insert, inventory_reservations insert; checkout flow then payment + ticket issuance. |

---

## BUYER (public + orders)

### GET /api/events (Public event listing)

| Field | Value |
|-------|--------|
| **Module** | Events (public) |
| **Endpoint** | GET /api/events |
| **Method** | GET |
| **Who calls it** | Anyone (buyers, unauthenticated) |
| **Purpose** | List published, public events with filters and pagination. |
| **Auth required** | No |
| **Query params** | `city`, `categoryId`, `tagIds` (comma or array), `dateFrom`, `dateTo`, `search`, `sort` (soonest \| newest \| popular), `page`, `pageSize` |
| **Response body** | `items` (array of event summaries: id, title, startAt, endAt, city, venueName, format, accessMode, category, tags, coverImageUrl, ticketsSold), `pagination` (page, pageSize, total) |
| **Error codes** | 500 INTERNAL_ERROR |
| **Business rules** | Only `status = 'published'` and `visibility_mode = 'public'`. |

---

### GET /api/events/:id (Public event detail)

| Field | Value |
|-------|--------|
| **Module** | Events (public) |
| **Endpoint** | GET /api/events/:id |
| **Method** | GET |
| **Who calls it** | Anyone (buyers) |
| **Purpose** | Get full event detail and public ticket types for a published public event. |
| **Auth required** | No |
| **Response body** | `event` (full detail: id, title, description, startAt, endAt, timezone, city, venueName, venueAddress, format, accessMode, category, tags, promoter, images), `ticketTypes` (id, name, description, currency, priceAmount, bookingFeeAmount, totalAmount, salesStartAt, salesEndAt, capacityTotal, capacitySold, capacityRemaining, perOrderLimit, status) |
| **Error codes** | 404 NOT_FOUND — event not found or not accessible |
| **Business rules** | Only published + public events. |

---

### GET /api/events/share/:shareToken (Private link event detail)

| Field | Value |
|-------|--------|
| **Module** | Events (public) |
| **Endpoint** | GET /api/events/share/:shareToken |
| **Method** | GET |
| **Who calls it** | Anyone with private link (buyers) |
| **Purpose** | Get event detail for a private-link event; sets access grant cookie for ordering. |
| **Auth required** | No |
| **Response body** | Same shape as GET /api/events/:id (event + ticketTypes) |
| **Error codes** | 404 NOT_FOUND — invalid token or not private_link |
| **Business rules** | Only published + visibility_mode = 'private_link'. Sets `event_access_grant` cookie for later order/checkout. |

---

### POST /api/orders/events/:eventId/orders

*(See full example above.)*

---

### POST /api/orders/:orderId/checkout (Complete order / payment)

| Field | Value |
|-------|--------|
| **Module** | Orders |
| **Endpoint** | POST /api/orders/:orderId/checkout |
| **Method** | POST |
| **Who calls it** | Buyer (order owner) |
| **Purpose** | Complete payment for an order; confirm tickets and release reservations. |
| **Auth required** | Yes. Role: order ownership enforced |
| **Request body** | (Payment provider–specific; e.g. payment intent confirmation) |
| **Response body** | Order confirmation, ticket IDs / confirmation reference |
| **Error codes** | 401 UNAUTHORIZED, 403 (not owner), 404 NOT_FOUND, 400 (already paid / expired / invalid state) |
| **Business rules** | Order must be in payable state; payment success triggers ticket creation and inventory finalisation. |

---

### GET /api/orders/me/orders, GET /api/orders/me/tickets, GET /api/orders/:orderId, GET /api/orders/:orderId/tickets, GET /api/orders/tickets/:ticketId, GET /api/orders/tickets/:ticketId/qr, POST /api/orders/:orderId/cancel

Buyer order and ticket management; not event CRUD but event-related (scoped by event via order). Omitted for brevity; same style applies.

---

## PROMOTER (event lifecycle + scanner)

Base path: **/api/promoters**. All require authenticated **promoter** and, where applicable, **event ownership**.

### POST /api/promoters/events (Create event)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | POST /api/promoters/events |
| **Method** | POST |
| **Who calls it** | Promoter (authenticated, active) |
| **Purpose** | Create a draft event with hierarchy (guru, network manager, territory) derived from promoter. |
| **Auth required** | Yes. Role: promoter; account_status: active |
| **Request body** | `title`, `description`, `startAt`, `endAt`, `timezone`, `format` (in_person \| online_live \| virtual_on_demand \| hybrid), `accessMode` (ticketed \| guest_list \| mixed), `visibilityMode` (public \| private_link), `city`, `venueName`, `venueAddress`, `lat`, `lng`, `categoryId`, `tagIds`, `tagNames` |
| **Response body** | `id` (eventId), `message` |
| **Error codes** | 400 VALIDATION_FAILED, 403 FORBIDDEN (inactive promoter), 500 INTERNAL_ERROR |
| **Business rules** | At least one of title/description/city/startAt/endAt required. If private_link, share_token generated. |
| **Calls internally** | deriveHierarchyFromPromoter(promoterId), events INSERT, attachTagsToEvent. |

---

### GET /api/promoters/events (List my events)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | GET /api/promoters/events |
| **Method** | GET |
| **Who calls it** | Promoter |
| **Purpose** | List events owned by the promoter (any status). |
| **Auth required** | Yes. Role: promoter |
| **Query params** | (pagination/status if implemented in controller) |
| **Response body** | List of promoter’s events |
| **Error codes** | 500 INTERNAL_ERROR |

---

### GET /api/promoters/events/:eventId (Event detail for promoter)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | GET /api/promoters/events/:eventId |
| **Method** | GET |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Full event detail for editing and management. |
| **Auth required** | Yes. Role: promoter; event ownership |
| **Response body** | Full event record (all fields) |
| **Error codes** | 404 NOT_FOUND, 500 INTERNAL_ERROR |

---

### PATCH /api/promoters/events/:eventId (Update event)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | PATCH /api/promoters/events/:eventId |
| **Method** | PATCH |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Update draft (or editable) event fields. |
| **Auth required** | Yes. Role: promoter; event ownership |
| **Request body** | Same fields as create (partial); optional `resetShareToken` for private_link |
| **Response body** | Success / updated event |
| **Error codes** | 400 INVALID_STATE (e.g. cancelled), 404 NOT_FOUND, 500 INTERNAL_ERROR |
| **Business rules** | Cancelled events cannot be edited. |

---

### DELETE /api/promoters/events/:eventId (Delete event)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | DELETE /api/promoters/events/:eventId |
| **Method** | DELETE |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Delete event (soft or hard per implementation). |
| **Auth required** | Yes. Role: promoter; event ownership |
| **Error codes** | 404 NOT_FOUND, 400 if not allowed (e.g. has orders), 500 INTERNAL_ERROR |

---

### POST /api/promoters/events/:eventId/images (Upload image)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | POST /api/promoters/events/:eventId/images |
| **Method** | POST |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Upload cover or gallery image (multipart). |
| **Auth required** | Yes. Role: promoter; event ownership |
| **Request body** | multipart: `image` file; body may indicate cover vs gallery |
| **Response body** | Image URL(s) / attachment info |
| **Error codes** | 400 VALIDATION_ERROR, 404 NOT_FOUND, 500 INTERNAL_ERROR |

---

### PUT /api/promoters/events/:eventId/category, PUT /api/promoters/events/:eventId/tags

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoints** | PUT /api/promoters/events/:eventId/category, PUT /api/promoters/events/:eventId/tags |
| **Method** | PUT |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Set event category or replace event tags. |
| **Auth required** | Yes. Role: promoter; event ownership |
| **Request body** | category: `categoryId`; tags: `tagIds` or `tagNames` |
| **Error codes** | 404 NOT_FOUND, 500 INTERNAL_ERROR |

---

### POST /api/promoters/events/:eventId/publish (Publish event)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | POST /api/promoters/events/:eventId/publish |
| **Method** | POST |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Move event from draft to published so it appears in public listing and can be purchased. |
| **Auth required** | Yes. Role: promoter; event ownership |
| **Request body** | (none) |
| **Response body** | `id`, `status` ('published'), `publishedAt` |
| **Error codes** | 400 INVALID_STATE (already published / cancelled), 400 VALIDATION_FAILED (missing required fields or at least one active ticket type for ticketed/mixed), 404 NOT_FOUND, 500 INTERNAL_ERROR |
| **Business rules** | Required: title, description, startAt, endAt, city, format, accessMode; if in_person/hybrid: venueName, venueAddress. If ticketed/mixed: at least one active ticket type. |
| **Calls internally** | logEventChange(req, 'published', eventId). |

---

### POST /api/promoters/events/:eventId/pause (Pause event)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | POST /api/promoters/events/:eventId/pause |
| **Method** | POST |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Unpublish event (hide from listing; no new sales). |
| **Auth required** | Yes. Role: promoter; event ownership |
| **Response body** | `id`, `status` ('unpublished' or equivalent) |
| **Error codes** | 400 INVALID_STATE, 404 NOT_FOUND, 500 INTERNAL_ERROR |

---

### POST /api/promoters/events/:eventId/cancel (Promoter cancel)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | POST /api/promoters/events/:eventId/cancel |
| **Method** | POST |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Cancel event (promoter-initiated). |
| **Auth required** | Yes. Role: promoter; event ownership |
| **Response body** | Success / status |
| **Error codes** | 400 INVALID_STATE, 404 NOT_FOUND, 500 INTERNAL_ERROR |

---

### POST /api/promoters/events/:eventId/republish (Republish event)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | POST /api/promoters/events/:eventId/republish |
| **Method** | POST |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Make a paused/unpublished event published again. |
| **Auth required** | Yes. Role: promoter; event ownership |
| **Error codes** | 400 INVALID_STATE, 404 NOT_FOUND, 500 INTERNAL_ERROR |

---

### POST /api/promoters/events/:eventId/complete (Promoter mark complete)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | POST /api/promoters/events/:eventId/complete |
| **Method** | POST |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Mark event as completed (lifecycle; rewards may be driven by admin complete in this codebase). |
| **Auth required** | Yes. Role: promoter; event ownership |
| **Error codes** | 404 NOT_FOUND, 400 INVALID_STATE, 500 INTERNAL_ERROR |

---

### DELETE /api/promoters/events/:eventId/cover, DELETE /api/promoters/events/:eventId/gallery/:imageId, PATCH /api/promoters/events/:eventId/images/reorder

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoints** | DELETE .../cover, DELETE .../gallery/:imageId, PATCH .../images/reorder |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Remove cover, remove gallery image, reorder gallery. |
| **Auth required** | Yes. Role: promoter; event ownership |

---

### GET /api/promoters/events/:eventId/performance (Event performance)

| Field | Value |
|-------|--------|
| **Module** | Events (promoter) |
| **Endpoint** | GET /api/promoters/events/:eventId/performance |
| **Method** | GET |
| **Who calls it** | Promoter (owner) |
| **Purpose** | Get sales/performance metrics for the event. |
| **Auth required** | Yes. Role: promoter; event ownership |
| **Response body** | Aggregates (e.g. tickets_sold, revenue) |
| **Error codes** | 404 NOT_FOUND, 500 INTERNAL_ERROR |

---

### Ticket types (promoter)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| GET /api/promoters/events/:eventId/ticket-types | GET | List ticket types for event |
| POST /api/promoters/events/:eventId/ticket-types | POST | Create ticket type |
| PATCH /api/promoters/ticket-types/:ticketTypeId | PATCH | Update ticket type |
| POST /api/promoters/ticket-types/:ticketTypeId/duplicate | POST | Duplicate ticket type |
| POST /api/promoters/ticket-types/:ticketTypeId/pause | POST | Pause ticket type |
| POST /api/promoters/ticket-types/:ticketTypeId/resume | POST | Resume ticket type |
| DELETE /api/promoters/ticket-types/:ticketTypeId | DELETE | Delete ticket type |

All require promoter + event/ticket-type ownership.

---

### Scanner / attendees (promoter)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| GET /api/promoters/events/:eventId/attendees | GET | List attendees for event |
| POST /api/promoters/events/:eventId/validate | POST | Validate a ticket (pre-check-in) |
| POST /api/promoters/events/:eventId/checkin | POST | Check-in a ticket |
| POST /api/promoters/events/:eventId/checkin/undo | POST | Undo check-in |
| GET /api/promoters/events/:eventId/logs | GET | Audit logs for event (scanner/check-in) |

All require promoter + event ownership.

---

### Access settings (promoter)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| GET /api/promoters/events/:eventId/access-settings | GET | Get access/scan settings |
| PATCH /api/promoters/events/:eventId/access-settings | PATCH | Update access settings |
| POST /api/promoters/events/:eventId/rotate-access | POST | Rotate access links/keys |

All require promoter + event ownership.

---

## ADMIN

Base path: **/api/admin**. All require **admin** role.

### GET /api/admin/events (List all events)

| Field | Value |
|-------|--------|
| **Module** | Events (admin) |
| **Endpoint** | GET /api/admin/events |
| **Method** | GET |
| **Who calls it** | Admin |
| **Purpose** | List all events with promoter info; filter by status and completion_status. |
| **Auth required** | Yes. Role: admin |
| **Query params** | `page`, `pageSize`, `status`, `completionStatus` |
| **Response body** | `events` (array: id, title, status, completion_status, tickets_sold, start_at, end_at, completed_at, completed_by, created_at, promoter_id, promoter_name, promoter_email), `pagination` |
| **Error codes** | 500 INTERNAL_ERROR |

---

### GET /api/admin/events/:eventId (Get event detail)

| Field | Value |
|-------|--------|
| **Module** | Events (admin) |
| **Endpoint** | GET /api/admin/events/:eventId |
| **Method** | GET |
| **Who calls it** | Admin |
| **Purpose** | Full event details and promoter info for any event. |
| **Auth required** | Yes. Role: admin |
| **Response body** | Event record + promoter name/email/id |
| **Error codes** | 404 NOT_FOUND, 500 INTERNAL_ERROR |

---

### POST /api/admin/events/:eventId/complete (Admin complete event)

| Field | Value |
|-------|--------|
| **Module** | Events (admin) |
| **Endpoint** | POST /api/admin/events/:eventId/complete |
| **Method** | POST |
| **Who calls it** | Admin |
| **Purpose** | Mark event as completed and trigger reward issuance (promoter/guru, etc.). |
| **Auth required** | Yes. Role: admin |
| **Request body** | (none) |
| **Response body** | `id`, `completionStatus` ('completed'), `completedAt`, `completedBy`, `rewardsIssued` (promoterReward, guruReward, ticketsSold or error) |
| **Error codes** | 404 NOT_FOUND, 400 INVALID_STATE (cancelled), 400 ALREADY_COMPLETED, 500 INTERNAL_ERROR |
| **Business rules** | Cancelled events cannot be completed. Rewards issued in separate flow; failure does not roll back completion. |
| **Calls internally** | logEventChange(req, 'admin_completed', eventId), issueRewardsForEvent, sendRewardNotificationEmails. |

---

### POST /api/admin/events/:eventId/cancel (Admin cancel event)

| Field | Value |
|-------|--------|
| **Module** | Events (admin) |
| **Endpoint** | POST /api/admin/events/:eventId/cancel |
| **Method** | POST |
| **Who calls it** | Admin |
| **Purpose** | Cancel event (admin-initiated); optional reason. |
| **Auth required** | Yes. Role: admin |
| **Request body** | `reason` (optional) |
| **Response body** | Success / event status |
| **Error codes** | 404 NOT_FOUND, 400 INVALID_STATE (e.g. already completed), 500 INTERNAL_ERROR |

---

### GET /api/admin/events/audit-logs (Event audit logs)

| Field | Value |
|-------|--------|
| **Module** | Events (admin) |
| **Endpoint** | GET /api/admin/events/audit-logs |
| **Method** | GET |
| **Who calls it** | Admin |
| **Purpose** | List event audit log entries (event changes, publish, etc.). |
| **Auth required** | Yes. Role: admin |
| **Query params** | `eventId`, `promoterId`, `action`, `page`, `pageSize` |
| **Response body** | `logs` (array), `pagination` |
| **Error codes** | 500 INTERNAL_ERROR |

---

### GET /api/admin/events/metrics (Event metrics dashboard)

| Field | Value |
|-------|--------|
| **Module** | Events (admin) |
| **Endpoint** | GET /api/admin/events/metrics |
| **Method** | GET |
| **Who calls it** | Admin |
| **Purpose** | Dashboard metrics (event counts by status, total views, etc.). |
| **Auth required** | Yes. Role: admin |
| **Response body** | statusMetrics, totalViews, etc. |
| **Error codes** | 500 INTERNAL_ERROR |

---

## Flow summary by role

- **Buyer:** Discover events (GET /api/events, GET /api/events/:id or share link) → create order (POST /api/orders/events/:eventId/orders) → checkout (POST /api/orders/:orderId/checkout) → view orders/tickets (GET /api/orders/me/orders, me/tickets, etc.).
- **Promoter:** Create/update event (POST/PATCH /api/promoters/events) → add ticket types and images → publish (POST .../publish); optionally pause/republish/cancel/complete; manage attendees (attendees, validate, check-in, logs) and access settings.
- **Admin:** List and inspect all events (GET /api/admin/events, GET /api/admin/events/:eventId) → complete event (POST /api/admin/events/:eventId/complete) or cancel (POST .../cancel); view audit logs and metrics (GET /api/admin/events/audit-logs, GET /api/admin/events/metrics).

---

*Generated from eventopia-stagging-backend codebase. Exact error codes and response shapes may vary slightly; refer to controllers and standardResponse for production.*
