# Auth APIs: Login & Register by Role

Base path: **`/api/auth`**

---

## One login for all roles

There is **one** email/password login endpoint. The **role** is stored on the user and returned in the response; the frontend uses it to route to the correct dashboard.

---

## 1. Register (email + password)

**Endpoint:** `POST /api/auth/register`

**Body (JSON):**

| Field | Required | Description |
|-------|----------|-------------|
| `email` | Yes | Valid email |
| `password` | Yes | Strong password (min 8 chars, upper, lower, number, special) |
| `role` or `role_requested` | No | `"buyer"` \| `"promoter"` \| `"guru"`. Defaults to `"buyer"` if omitted |
| `city` | No | City (e.g. for preferences) |
| `deviceId` | No | Optional device id |
| `guruCode` | No | Referral code from Guru link |
| `invite_token` | No | Guru invite token (from admin-created invite) |

**Role behaviour:**

| You pass | What happens |
|----------|--------------|
| Nothing or `role: "buyer"` | User created with `role: "buyer"`, `account_status: "active"`. Can log in immediately after email verification. |
| `role: "promoter"` | User created with `role: "promoter"`, `account_status: "pending"`. Must verify email (OTP). Cannot log in until admin approves (`account_status: "active"`). |
| `role: "guru"` | User created with `role: "buyer"` initially, `account_status: "pending"`. Must verify email (OTP). After approval, role becomes guru. Or use **invite_token** (see below). |
| `role: "network_manager"` | User created with `role: null`, `account_status: "requested"`. Must verify email. Cannot self-register as network_manager for login; admin approves and sets role. |

**With invite (Guru):**  
If you pass a valid `invite_token` (and email matches the invite), the invite’s role is used: user may be created as `buyer` with a guru application pending, and must verify email with OTP.

**Response (201):**  
Returns `email`, `userId`, `challengeId`, `expires-in`, and often an OTP in the payload (for dev). For **buyer** (no approval flow), you get a prompt to verify email; for **promoter/guru/network_manager** you get the same, plus `roleRequested` or `invited: true` etc.

**Next step after register:**  
Call `POST /api/auth/otp/verify` with `{ "userId", "otp" }` (using the `challengeId` from register if your flow uses it) to verify email. Then use **login**.

---

## 2. Login (email + password) – all roles

**Endpoint:** `POST /api/auth/login`

**Body (JSON):**

| Field | Required | Description |
|-------|----------|-------------|
| `email` | Yes | Registered email |
| `password` | Yes | Password |
| `deviceId` | No | Optional |

**Response (200):**

```json
{
  "error": false,
  "message": "You have logged in successfully.",
  "data": {
    "email": "user@example.com",
    "userId": 123,
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "expires-at": "2025-02-06T12:30:00.000Z",
    "setupRequired": false,
    "accountStatus": "active",
    "emailStatus": "verified",
    "role": "buyer",
    "user": {
      "userId": 123,
      "email": "user@example.com",
      "name": "...",
      "role": "buyer",
      "emailStatus": "verified"
    }
  }
}
```

**How roles affect login:**

- **buyer:** Can log in as soon as `account_status` is active (default after register). `role` in response is `"buyer"`.
- **promoter:** Can log in only when `account_status === "active"` (after admin approval). Response `role` is `"promoter"`.
- **guru:** Same: must be approved (`account_status: "active"`). Response `role` is `"guru"`.
- **network_manager:** User must have `role: "network_manager"` (set by admin). If `role` is null (applicant), login is blocked with “pending approval”.

**Frontend:** Use `data.role` to route (e.g. buyer → Event Discovery, promoter → Promoter dashboard, guru → Guru dashboard, admin → Admin).

---

## 3. Get current user (after login)

**Endpoint:** `GET /api/auth/me`  
**Headers:** `Authorization: Bearer <accessToken>`

Returns full profile, including `role`, `accountStatus`, and optional `networkManagerApplication`, `guruApplication`, `promoterApplication` so the frontend can show the right UI (e.g. pending approval, setup required).

---

## 4. King’s Account (separate flow)

King’s login does **not** use the main login endpoint. It uses OTP only.

**Register (Kings):**  
`POST /api/auth/king/register`  
**Body:** `{ "email", "password" }`  
- Creates user with `role: "kings_account"`.  
- Disabled in production (`NODE_ENV=production`).

**Send OTP:**  
`POST /api/auth/king/otp/send`  
**Body:** `{ "email": "kings.test@eventopia.local" }`  
- Sends OTP to email.  
- **Response:** `challengeId`, `expires-in` (seconds), and in dev the `otp` code.

**Verify OTP (login):**  
`POST /api/auth/king/otp/verify`  
**Body:** `{ "email", "otp", "challengeId" }`  
- On success returns `accessToken`, `refreshToken`, `role: "kings_account"`.

---

## 5. OAuth register (optional)

**Endpoint:** `POST /api/auth/oauth/register`

**Body:** `email`, `name`, `oauthProvider`, `oauthId`, `avatarUrl`, `role` or `role_requested`, optional `applicationData`.  
Same role rules as email register: buyer/promoter/guru self-select; network_manager is special (e.g. applicant until approved).

---

## 6. Quick reference: what to pass per role

| Role | Register body | Login | When they can log in |
|------|----------------|-------|------------------------|
| **Buyer** | `email`, `password` (optional: `role: "buyer"`, `city`) | `email`, `password` | After email verification; `account_status` is active by default |
| **Promoter** | `email`, `password`, `role: "promoter"` | `email`, `password` | After email verification **and** admin approval |
| **Guru** | `email`, `password`, `role: "guru"` or `invite_token` | `email`, `password` | After email verification **and** admin approval (and any activation flow) |
| **Network manager** | Not self-registered as NM; or register with `role: "network_manager"` (creates applicant) | `email`, `password` | Only after admin sets `role: "network_manager"` and account is active |
| **King’s** | `POST /king/register` then `/king/otp/send` then `/king/otp/verify` | No password login; use OTP verify as “login” | After OTP verify |

---

## 7. Token usage

- Send **accessToken** in header: `Authorization: Bearer <accessToken>` on all protected routes.
- When the access token expires, call `POST /api/auth/refresh` with body `{ "refresh_token": "<refreshToken>" }` to get a new access token.
- Role is stored in the token/session; backend middleware (`requireRole`, `requireActivePromoter`, etc.) blocks access by role (e.g. buyer cannot call guru-only APIs).
