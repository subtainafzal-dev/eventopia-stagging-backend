# Money Splitting Flow (Escrow + Credit Engine)

This document describes the current backend implementation of money splitting in the ticket purchase flow, including exact tables used at each step.

## Scope

- Current flow is based on hardcoded payment intent/reference and confirm endpoint.
- Stripe integration is pending.
- This is an as-implemented reference for PM/engineering alignment.

## Stripe Integration View (How money will move to tables)

When Stripe is integrated, the table movement should stay mostly the same, but the payment trigger source changes from manual confirm to Stripe webhook/verified callback.

### Target payment lifecycle with Stripe

1. Create order in pending state
2. Create Stripe PaymentIntent (store Stripe intent id on order)
3. Buyer pays on Stripe Checkout/PaymentIntent flow
4. Stripe sends webhook (`payment_intent.succeeded` or failure event)
5. Backend verifies signature + idempotency
6. Backend confirms order and runs financial postings

### Stripe-driven table movement (step-wise)

#### Step S1: Create order and attach Stripe intent

- `orders`
  - Insert pending order (same as now)
  - Update/store real Stripe values:
    - `payment_intent_id = <stripe_pi_id>`
    - `payment_provider = stripe`
    - `status = payment_pending`
    - `payment_status = unpaid`
- `order_items`
  - Per-ticket amounts and attendee-level pricing
- `inventory_reservations`
  - Reserve stock while payment is pending

#### Step S2: Stripe webhook received (success/failure)

- `webhook_events`
  - Insert provider event id for idempotency (one-time processing guard)
- `orders`
  - If success: move to confirmed/paid
  - If failed: move to failed/unpaid state
- `inventory_reservations`
  - Success: `active -> consumed`
  - Failure: `active -> cancelled`

#### Step S3: On Stripe success -> mint tickets and sales updates

- `tickets`
  - Insert minted tickets per attendee
- `ticket_types`
  - Increment `qty_sold`
- `events`
  - Recompute/update `tickets_sold`

#### Step S4: Escrow money posting (ticket subtotal)

- `escrow_accounts`
  - Add subtotal to escrow balance (`balance`)
- `escrow_liabilities`
  - Increase `gross_ticket_revenue` for event/promoter liability
- `ledger_entries`
  - Insert escrow audit entry: `entry_type = ESCROW_RECEIVE`

#### Step S5: Booking fee posting

- `ledger_entries`
  - Insert booking fee audit entry: `entry_type = BOOKING_FEE`
- Source values:
  - `orders.booking_fee_amount`
  - `order_items.ticket_booking_fee_amount`
- Current design note:
  - Booking fee remains ledger-tracked (no separate operating-balance table update in this flow)

#### Step S6: Credit engine posting (tier split)

- `credit_ledger`
  - Insert allocation rows (`entry_type = CREDIT_ALLOCATION`, projected metadata)
- `credit_wallets`
  - Increase `projected_balance` for beneficiary role wallets
- `ledger_entries`
  - Insert allocation audit entry: `entry_type = CREDIT_ALLOCATION`

### Stripe events to map in implementation

- `payment_intent.succeeded`
  - Run S2 success path + S3/S4/S5/S6
- `payment_intent.payment_failed`
  - Run S2 failure path (cancel reservation, mark order failed)
- Optional refund events (future)
  - Should reverse/adjust in:
    - `escrow_liabilities` (refund deductions)
    - `ledger_entries` (refund entries)
    - `credit_wallets`/`credit_ledger` (if credit reversal policy applies)

## Complete Table List In This Money-Splitting Flow (Step-wise)

### Step 1: Buyer creates order (payment pending)

- `orders`
  - Stores `subtotal_amount`, `booking_fee_amount`, `total_amount`, `payment_intent_id`, `payment_provider`, `status`, `payment_status`.
- `order_items`
  - Stores per-ticket values: `ticket_price_amount`, `ticket_booking_fee_amount`, `subtotal_amount`, attendee info.
- `inventory_reservations`
  - Reserves inventory before payment confirmation (`status = active`).

### Step 2: Payment confirm trigger

- `orders`
  - Updated to confirmed/paid (`status`, `payment_status`, `confirmed_at`).
- `inventory_reservations`
  - `active -> consumed` on successful payment.
- `ticket_types`
  - `qty_sold` increments.
- `tickets`
  - Ticket rows are minted/inserted per attendee.
- `events`
  - `tickets_sold` updated from active ticket count.

### Step 3: Escrow posting (ticket subtotal flow)

- `escrow_accounts`
  - Escrow amount (ticket subtotal) is added to escrow balance (`balance`).
- `escrow_liabilities`
  - Event/promoter liability updated (`gross_ticket_revenue` incremented).
- `ledger_entries`
  - Audit entry created with `entry_type = ESCROW_RECEIVE`.

### Step 4: Booking fee posting

- `ledger_entries`
  - Booking fee is recorded as `entry_type = BOOKING_FEE`.
- Source values come from:
  - `orders.booking_fee_amount`
  - `order_items.ticket_booking_fee_amount`
- Note:
  - No separate operating-account balance table is used in this purchase path.

### Step 5: Credit engine split posting (tier-based projected credit)

- `credit_ledger`
  - Immutable credit allocation records (`entry_type = CREDIT_ALLOCATION`, projected metadata).
- `credit_wallets`
  - `projected_balance` increases for role wallets (promoter/guru/network_manager where applicable).
- `ledger_entries`
  - Financial audit row also created with `entry_type = CREDIT_ALLOCATION`.

### Step 6: Optional webhook idempotency (when webhook route is used)

- `webhook_events`
  - Stores processed provider event IDs to prevent duplicate processing.

## One-line Money Path (for presentation)

Buyer pays -> `orders` stores totals -> confirm -> subtotal goes to `escrow_accounts.balance`, booking_fee goes to `ledger_entries` as `BOOKING_FEE`, tier split goes to `credit_ledger` + `credit_wallets.projected_balance` + `ledger_entries` as `CREDIT_ALLOCATION`.

Stripe version: Buyer pays on Stripe -> webhook verifies and marks `orders` paid -> same downstream postings happen: subtotal to `escrow_accounts.balance`, booking fee to `ledger_entries` (`BOOKING_FEE`), tier split to `credit_ledger` + `credit_wallets.projected_balance` + `ledger_entries` (`CREDIT_ALLOCATION`).

## Important Implementation Notes

- Booking fee is currently ledger-only in this flow (no dedicated operating balance table update).
- Credit allocation in this flow is projected credit, not direct cash payout.
- Amount units are mostly in pence for order/ledger/credit writes; escrow liability uses numeric currency values.
