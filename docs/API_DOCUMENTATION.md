# Eventopia API Documentation

Base URL: `/api` (e.g. `http://localhost:3000/api`).  
Protected routes use header: `Authorization: Bearer <access_token>`.

Standard response shape:
- Success: `{ "error": false, "data": { ... }, "request_id": "..." }`
- Error: `{ "error": true, "code": "ERROR_CODE", "message": "...", "data": null }`

---

## 0. Health & Public (no auth)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 1 | GET | `/health` (root: `http://localhost:3000/health`) | Server health. Returns `status`, `uptime`, `version`. |
| 2 | GET | `/api/events` | List published public events. |
| 3 | GET | `/api/events/:id` | Get single public event with ticket types. |
| 4 | GET | `/api/events/share/:shareToken` | Get private-link event; sets access cookie. |

### GET /api/events (query)

- **Query:** `page`, `pageSize`, `sort` (soonest \| newest \| popular), `city`, `categoryId`, `tagIds`, `dateFrom`, `dateTo`, `search`
- **Response:** `{ "items": [{ "id", "title", "startAt", "endAt", "city", "venueName", "format", "accessMode", "category", "tags", "coverImageUrl", "ticketsSold" }], "pagination": { "page", "pageSize", "total" } }`

### GET /api/events/:id (response)

- **Response:** `{ "event": { "id", "title", "description", "startAt", "endAt", "timezone", "city", "venueName", "venueAddress", "format", "accessMode", "category", "tags", "promoter", "images" }, "ticketTypes": [{ "id", "name", "description", "currency", "priceAmount", "bookingFeeAmount", "totalAmount", "capacityTotal", "capacitySold", "capacityRemaining", "perOrderLimit", "status" }] }`

---

## 1. Auth

| # | Method | Endpoint | Auth | Description |
|---|--------|----------|------|-------------|
| 5 | POST | `/api/auth/register` | No | Email/password registration |
| 6 | POST | `/api/auth/login` | No | Login; returns access + refresh token |
| 7 | POST | `/api/auth/refresh` | No | New access token from refresh token |
| 8 | GET | `/api/auth/me` | Yes | Current user |
| 9 | POST | `/api/auth/logout` | Yes | Logout current session |
| 10 | POST | `/api/auth/logout-all` | Yes | Logout all sessions |
| 11 | POST | `/api/auth/verify-email` | No | Verify email with token |
| 12 | POST | `/api/auth/forgot-password` | No | Request password reset email |
| 13 | POST | `/api/auth/reset-password` | No | Reset password with token |
| 14 | PATCH | `/api/auth/me` | Yes | Update profile |
| 15 | POST | `/api/auth/me/active-role` | Yes | Set active role |
| 16 | POST | `/api/auth/otp/verify` | No | OTP verify (registration) |
| 17 | POST | `/api/auth/otp/resend` | No | Resend OTP |

### POST /api/auth/register

- **Body:** `{ "email": "string", "password": "string", "name": "string" }`
- **Response:** User + tokens or verification required

### POST /api/auth/login

- **Body:** `{ "email": "string", "password": "string" }`
- **Response:** `{ "accessToken", "refreshToken", "user": { "id", "email", "name", "role", "account_status" } }`

### POST /api/auth/refresh

- **Body:** `{ "refresh_token": "string" }`
- **Response:** `{ "accessToken", "refreshToken" }`

### GET /api/auth/me

- **Headers:** `Authorization: Bearer <token>`
- **Response:** `{ "id", "email", "name", "role", "account_status", "roles_version", ... }`

---

## 2. Buyer – Orders (auth required)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 18 | POST | `/api/orders/events/:eventId/orders` | Create order |
| 19 | POST | `/api/orders/:orderId/checkout` | Checkout / complete payment |
| 20 | GET | `/api/orders/me/orders` | My orders |
| 21 | GET | `/api/orders/me/tickets` | My tickets |
| 22 | GET | `/api/orders/:orderId` | Order details (owner) |
| 23 | GET | `/api/orders/:orderId/tickets` | Order tickets |
| 24 | GET | `/api/orders/tickets/:ticketId` | Ticket details |
| 25 | GET | `/api/orders/tickets/:ticketId/qr` | Ticket QR payload |
| 26 | POST | `/api/orders/:orderId/cancel` | Cancel order |

### POST /api/orders/events/:eventId/orders

- **Body:** `{ "idempotencyKey": "string (required)", "items": [{ "ticketTypeId": number, "quantity": number, "buyerName?", "buyerEmail?" }] }`
- **Response:** `{ "orderId", "status", "totalAmount", "currency", "expiresAt" }`
- **Errors:** 400 EVENT_NOT_PUBLISHED, 404 TICKET_TYPE_NOT_FOUND, 400 TICKET_TYPE_NOT_ACTIVE, 403 ACCESS_DENIED (private link)

### POST /api/orders/:orderId/checkout

- **Response:** Order confirmation / ticket issuance (structure depends on payment flow)

### GET /api/orders/me/orders

- **Response:** `{ "orders": [{ "id", "order_number", "event_id", "total_amount", "status", "payment_status", "created_at" }] }`

---

## 3. Promoter – Events (auth: promoter)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 27 | POST | `/api/promoters/events` | Create event |
| 28 | GET | `/api/promoters/events` | List my events |
| 29 | GET | `/api/promoters/events/:eventId` | Event detail (owner) |
| 30 | PATCH | `/api/promoters/events/:eventId` | Update event |
| 31 | DELETE | `/api/promoters/events/:eventId` | Delete event |
| 32 | POST | `/api/promoters/events/:eventId/images` | Upload image (multipart) |
| 33 | PUT | `/api/promoters/events/:eventId/category` | Set category |
| 34 | PUT | `/api/promoters/events/:eventId/tags` | Set tags |
| 35 | POST | `/api/promoters/events/:eventId/publish` | Publish event |
| 36 | POST | `/api/promoters/events/:eventId/pause` | Pause (unpublish) |
| 37 | POST | `/api/promoters/events/:eventId/cancel` | Cancel event |
| 38 | POST | `/api/promoters/events/:eventId/republish` | Republish |
| 39 | POST | `/api/promoters/events/:eventId/complete` | Mark complete |
| 40 | GET | `/api/promoters/events/:eventId/performance` | Event performance |

### POST /api/promoters/events (body)

- **Body:** `title`, `description`, `startAt`, `endAt`, `timezone`, `format` (in_person \| online_live \| virtual_on_demand \| hybrid), `accessMode` (ticketed \| guest_list \| mixed), `visibilityMode` (public \| private_link), `city`, `venueName`, `venueAddress`, `lat`, `lng`, `categoryId`, `tagIds`, `tagNames`
- **Response:** `{ "id": eventId, "message": "Event created successfully" }`

### POST /api/promoters/events/:eventId/publish

- **Response:** `{ "id", "status": "published", "publishedAt" }`
- **Errors:** 400 VALIDATION_FAILED (missing fields or no active ticket type)

---

## 4. Promoter – Ticket types (auth: promoter, event owner)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 41 | GET | `/api/promoters/events/:eventId/ticket-types` | List ticket types |
| 42 | POST | `/api/promoters/events/:eventId/ticket-types` | Create ticket type |
| 43 | PATCH | `/api/promoters/ticket-types/:ticketTypeId` | Update ticket type |
| 44 | POST | `/api/promoters/ticket-types/:ticketTypeId/duplicate` | Duplicate |
| 45 | POST | `/api/promoters/ticket-types/:ticketTypeId/pause` | Pause |
| 46 | POST | `/api/promoters/ticket-types/:ticketTypeId/resume` | Resume |
| 47 | DELETE | `/api/promoters/ticket-types/:ticketTypeId` | Delete |

### POST /api/promoters/events/:eventId/ticket-types (body)

- **Body:** `name`, `description`, `priceAmount` (pence), `bookingFeeAmount` (pence), `capacityTotal`, `perOrderLimit`, `salesStartAt`, `salesEndAt`, `visibility` (public \| hidden)
- **Response:** Created ticket type object

---

## 5. Promoter – Attendees & scanner (auth: promoter, event owner)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 48 | GET | `/api/promoters/events/:eventId/attendees` | List attendees |
| 49 | POST | `/api/promoters/events/:eventId/validate` | Validate ticket |
| 50 | POST | `/api/promoters/events/:eventId/checkin` | Check-in ticket |
| 51 | POST | `/api/promoters/events/:eventId/checkin/undo` | Undo check-in |
| 52 | GET | `/api/promoters/events/:eventId/logs` | Event logs |

---

## 6. Promoter – Charity (auth: promoter)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 53 | POST | `/api/promoters/charity/applications` | Create charity application |
| 54 | GET | `/api/promoters/charity/applications` | List my applications |
| 55 | GET | `/api/promoters/charity/applications/:id` | Get application |
| 56 | PUT | `/api/promoters/charity/applications/:id` | Update (draft) |
| 57 | POST | `/api/promoters/charity/applications/:id/submit` | Submit |
| 58 | POST | `/api/promoters/charity/applications/:id/pay-fee` | Pay application fee |
| 59 | GET | `/api/promoters/charity/applications/:id/status` | Status |
| 60 | GET | `/api/promoters/charity/applications/:id/executions` | Executions (read-only) |

### POST /api/promoters/charity/applications (body)

- **Body:** `charity_name`, `charity_number`, `charity_description`, `charitable_objectives`, `requested_amount` (pence), `event_id?`, `charity_website?`, `beneficiary_details?`
- **Response:** `{ "application": { "id", "status": "DRAFT", ... } }`

### POST /api/promoters/charity/applications/:id/pay-fee (body)

- **Body:** `{ "idempotency_key": "string" }`
- **Response:** `{ "payment": { "payment_id", "payment_intent_id", "amount", "redirect_url" } }`

---

## 7. Admin – Events (auth: admin)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 61 | GET | `/api/admin/events` | List all events |
| 62 | GET | `/api/admin/events/:eventId` | Get event |
| 63 | POST | `/api/admin/events/:eventId/complete` | Complete event |
| 64 | POST | `/api/admin/events/:eventId/cancel` | Cancel event |
| 65 | GET | `/api/admin/events/audit-logs` | Event audit logs |
| 66 | GET | `/api/admin/events/metrics` | Event metrics |

### GET /api/admin/events (query)

- **Query:** `page`, `pageSize`, `status`, `completionStatus`
- **Response:** `{ "events": [...], "pagination": { "page", "pageSize", "total" } }`

### POST /api/admin/events/:eventId/cancel (body)

- **Body:** `{ "reason": "string (optional)" }`

---

## 8. Admin – Charity (auth: admin)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 67 | GET | `/api/admin/charity/applications` | List applications |
| 68 | GET | `/api/admin/charity/applications/:id` | Get application + payments/decisions/executions |
| 69 | POST | `/api/admin/charity/applications/:id/approve` | Approve |
| 70 | POST | `/api/admin/charity/applications/:id/partial-approve` | Partial approve |
| 71 | POST | `/api/admin/charity/applications/:id/reject` | Reject |
| 72 | POST | `/api/admin/charity/applications/:id/execute` | Create execution (payout) |
| 73 | PATCH | `/api/admin/charity/executions/:id` | Mark execution completed |
| 74 | POST | `/api/admin/charity/applications/:id/complete` | Mark application completed |
| 75 | GET | `/api/admin/charity/ledger` | Charity ledger |
| 76 | GET | `/api/admin/charity/balance` | Charity pot balance |

### POST /api/admin/charity/applications/:id/approve (body)

- **Body:** `{ "decision_amount": number (pence), "admin_notes": "string" }`

### POST /api/admin/charity/applications/:id/execute (body)

- **Body:** `recipient_type` (venue \| supplier \| marketing_platform), `recipient_name`, `amount` (pence), `execution_reference?`, `recipient_details?`

---

## 9. Admin – Gurus & Promoters (auth: admin)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 77 | GET | `/api/admin/gurus` | List gurus |
| 78 | GET | `/api/admin/gurus/:guruId` | Guru details |
| 79 | POST | `/api/admin/gurus/create-invite` | Create guru invite |
| 80 | POST | `/api/admin/gurus/:guruId/activate` | Activate guru |
| 81 | POST | `/api/admin/gurus/:guruId/level` | Update guru level |
| 82 | POST | `/api/admin/gurus/:applicationId/approve` | Approve guru application |
| 83 | GET | `/api/admin/promoters` | List promoters |
| 84 | GET | `/api/admin/promoters/:promoterId` | Promoter details |
| 85 | POST | `/api/admin/promoters/:applicationId/approve` | Approve promoter application |

---

## 10. Admin – Territories & King's Account (auth: admin)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 86 | GET | `/api/admin/territories` | List territories |
| 87 | GET | `/api/admin/territories/:id` | Territory details |
| 88 | GET | `/api/admin/territories/:id/licences` | Territory licences |
| 89 | POST | `/api/admin/territories` | Create territory |
| 90 | PATCH | `/api/admin/territories/:id` | Update territory |
| 91 | GET | `/api/admin/territory-applications` | List territory applications |
| 92 | POST | `/api/admin/territory-applications/:id/approve` | Approve |
| 93 | POST | `/api/admin/territory-applications/:id/reject` | Reject |
| 94 | POST | `/api/admin/territory-licences/:id/suspend` | Suspend licence |
| 95 | GET | `/api/admin/kings-account/overview` | King's Account overview |
| 96 | GET | `/api/admin/ledger` | Platform ledger |
| 97 | GET | `/api/admin/obligations` | Obligations |
| 98 | GET | `/api/admin/signup-fees` | Signup fees |
| 99 | GET | `/api/admin/exports/ledger.csv` | Export ledger CSV |
| 100 | GET | `/api/admin/exports/obligations.csv` | Export obligations CSV |
| 101 | GET | `/api/admin/exports/pots.csv` | Export pots CSV |
| 102 | GET | `/api/admin/exports/signup-fees.csv` | Export signup fees CSV |

---

## 11. Admin – Health & Audit (auth: admin)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 103 | GET | `/api/admin/health/summary` | Health summary |
| 104 | GET | `/api/admin/health/jobs` | Job runs |
| 105 | GET | `/api/admin/audit` | Audit logs |

---

## 12. Ledger Core – King's Account (auth: kings_account only)

These endpoints provide the immutable audit ledger for regulators. **Only users with the `kings_account` role** can call them. Others receive **403 WRONG_ROLE**.

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| L1 | GET | `/api/v1/ledger` | Paginated list of ledger entries (optional filters) |
| L2 | GET | `/api/v1/ledger/export` | Download ledger as CSV (requires territory_id, from, to; max 90 days) |
| L3 | GET | `/api/v1/ledger/:entry_id` | Full detail of one ledger entry |

### GET /api/v1/ledger

- **Query:** `territory_id` (optional), `entry_type` (optional), `user_id` (optional), `from` (ISO date), `to` (ISO date), `page` (default 1), `limit` (default 50, max 100).
- **Response:** `{ "data": { "entries": [ { "id", "entry_type", "user_id", "role", "level", "territory_id", "network_id", "amount", "rate_applied", "gross_credit", "net_credit", "reference_id", "reference_type", "approval_actor_id", "proof_reference", "status", "created_at" } ], "total", "page", "limit" } }`.
- **Sort:** `created_at` descending.

### GET /api/v1/ledger/:entry_id

- **Response:** Single entry with all 16 fields (null when not set). **404 NOT_FOUND** if entry does not exist.

### GET /api/v1/ledger/export

- **Query (required):** `territory_id`, `from` (ISO date), `to` (ISO date).
- **Rules:** Date range must not exceed **90 days**; otherwise **400 DATE_RANGE_TOO_LARGE**.
- **Response:** CSV file download with all 16 ledger columns. Before returning, the server records a **LEDGER_EXPORT** entry in the ledger (audit trail of who exported and when).

---

## 13. Network Manager (auth: network_manager)

| # | Method | Endpoint | Description    |
|---|--------|----------|-------------|
| 106 | GET | `/api/network/territories` | Territories |
| 107 | POST | `/api/network/territories/:territoryId/reserve` | Reserve territory |
| 108 | POST | `/api/network/territories/:territoryId/apply` | Apply for territory |
| 109 | GET | `/api/network-managers/gurus` | List gurus |
| 110 | GET | `/api/network-managers/gurus/applications` | Guru applications |
| 111 | POST | `/api/network-managers/gurus/:applicationId/approve` | Approve guru |
| 112 | POST | `/api/network-managers/gurus/:applicationId/reject` | Reject guru |
| 113 | GET | `/api/network-managers/dashboard/summary` | Dashboard summary |

---

## 14. Guru (auth: guru)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 114 | GET | `/api/gurus/dashboard/summary` | Dashboard summary |
| 115 | GET | `/api/gurus/dashboard/promoters` | Attached promoters |
| 116 | GET | `/api/gurus/promoters/applications` | Promoter applications |
| 117 | POST | `/api/gurus/promoters/:applicationId/approve` | Approve promoter |
| 118 | POST | `/api/gurus/promoters/:applicationId/reject` | Reject promoter |
| 119 | GET | `/api/gurus/rewards` | My rewards |

---

## 15. Categories & Tags (public or auth as needed)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 120 | GET | `/api/categories` | List categories |
| 121 | GET | `/api/categories/tree` | Categories tree |
| 122 | GET | `/api/categories/:id` | Category by ID |
| 123 | POST | `/api/categories` | Create (admin) |
| 124 | GET | `/api/tags` | List tags |
| 125 | GET | `/api/tags/:id` | Tag by ID |

---

## 16. Rewards & User preferences (auth)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 126 | GET | `/api/rewards/balance` | Reward balance |
| 127 | POST | `/api/rewards/redemptions` | Create redemption |
| 128 | GET | `/api/rewards/redemptions` | My redemptions |
| 129 | GET | `/api/users/me/preferences` | User preferences |
| 130 | PUT | `/api/users/me/preferences` | Update preferences |
| 131 | PATCH | `/api/users/me/change-password` | Change password |

---

## 17. Ticket access (auth as needed)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 132 | GET | `/api/tickets/:ticketId/access` | Ticket access state |
| 133 | POST | `/api/tickets/:ticketId/access-session` | Create access session |
| 134 | GET | `/api/access/:token` | Resolve access token |

---

## 18. Webhooks (external; no Bearer auth)

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 135 | POST | `/api/webhooks/payment` | Payment webhook |
| 136 | POST | `/api/webhooks/charity-payment` | Charity payment webhook |

---

## Quick reference: by role

- **Public/Buyer:** 0 (Health & Public), 1 (Auth login/register), 2 (Orders), 10 (Categories/Tags), 14 (Rewards/Preferences).
- **Promoter:** 1 (Auth), 3 (Events), 4 (Ticket types), 5 (Attendees), 6 (Charity).
- **Admin:** 1 (Auth), 7 (Admin Events), 8 (Admin Charity), 9 (Gurus/Promoters), 10 (Territories/King's Account), 11 (Health/Audit).
- **King's Account:** 1 (Auth), 12 (Ledger Core – list, export, entry detail).
- **Network Manager:** 1 (Auth), 12 (Network Manager).
- **Guru:** 1 (Auth), 13 (Guru).
