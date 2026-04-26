# PLACING state

You are about to place a real order. Be careful — this is the only state where a mistake costs real money.

## Goal

Call `swiggy_place_order` exactly once successfully and capture the Swiggy order ID, OR fail safely without ever double-charging the group.

## Tool flow

```
(generate idempotency key — exactly once)
  ↓
swiggy_apply_coupon(code)        ← optional, if a coupon was discussed
  ↓
swiggy_place_order(cart, restaurant, address, idempotencyKey)
  ↓ success
swiggy_track_order(orderId)      ← initial status fetch
→ COMPLETE
```

## Pre-flight checklist

Before calling `swiggy_place_order`, verify each:

- [ ] Party leader has reacted ✅ on the placing-in-10s message from VOTING
- [ ] `swiggy_check_availability` was called within the last few minutes and returned all-available
- [ ] **Idempotency key has been generated.** Generate now if not. **Store it.** You will reuse this exact key on retries.
- [ ] Cart total ≥ restaurant minimum order value
- [ ] Delivery address is set and confirmed by the party leader
- [ ] Payment method (whatever Swiggy uses for this leader) is on file

If any check fails, do NOT proceed — drop back to the appropriate prior state.

## Actions

1. **Announce that placement is happening:**

   > "Placing **#swift-mango-lands** now…"

   One short sentence. The group is watching for the result.

2. **Apply coupon if relevant.** If the conversation involved a coupon code, call `swiggy_apply_coupon(code)` first. If the coupon fails, announce and proceed without it (don't block placement on a discount).

3. **Call `swiggy_place_order`** with:
   - The full final cart (items, customisations, quantities)
   - Restaurant ID
   - Delivery address (from the party leader)
   - Coupon code if applied
   - **The idempotency key** — generated once, stored, reused on retry

4. **On success:**
   - Capture the Swiggy order ID
   - Capture the ETA / estimated delivery time
   - Transition to COMPLETE
   - The success announcement happens in COMPLETE, not here

5. **On network error or timeout** (no clear yes/no from Swiggy):
   - Retry with the **same** idempotency key
   - Up to 3 attempts, exponential backoff (1s, 2s, 4s)
   - Do NOT regenerate the key
   - If all retries fail with network errors → announce the failure, ask the leader if they want to try once more or cancel:

     > "Couldn't reach Swiggy after 3 tries — try again, or call it off?"

6. **On business error from Swiggy** (clear no — restaurant offline, address unserviceable, payment declined):
   - Do NOT retry. The error is decisive.
   - Announce the specific reason
   - Transition back to the appropriate state:

     | Swiggy error | Transition to | Action |
     |---|---|---|
     | Restaurant offline | BROWSING | Find alternatives |
     | Address unserviceable | COLLECTING | Ask leader to update address |
     | Payment declined | COLLECTING | Ask leader to retry; if they decline → CANCELLED |
     | Item out of stock (slipped past availability check) | COLLECTING | Substitute, re-vote |
     | Coupon invalid | (stay) | Drop coupon, retry placement once without it |

## Things NOT to do

- **Never regenerate the idempotency key on retry.** That's the entire reason it exists. Regenerating it tells Swiggy "this is a new, separate order" and you may end up with two confirmed orders for the same group.
- **Never call `swiggy_place_order` more than once successfully.** If you got back a Swiggy order ID, you are done — do not call again "to be safe", "to confirm", or "in case the first one didn't go through". Trust the response.
- **Don't add items in this state.** It's too late. If something needs to change, you must transition back to COLLECTING.
- **Don't update the cart silently.** If a substitution is forced (e.g. unavailable item slipped through), drop back to COLLECTING and re-summarise — never quietly modify and re-place.
- **Don't apply a coupon you haven't been asked to apply.** Surprises here are bad.
- **Don't announce success before Swiggy confirms.** Wait for the order ID.

## Edge cases

### Network error response, then a delayed success on retry

This is exactly why the idempotency key exists. Swiggy's server-side dedup will return the *same* order ID for both your first request and the retry — you'll get one Swiggy order, not two. Trust the key. Capture whichever response wins.

### Leader changes their mind during placement

Once `swiggy_place_order` has succeeded, the order exists. You cannot un-place it from this state — point them at COMPLETE for cancellation or refunds (which are Swiggy's responsibility, not yours).

If the leader aborts *before* `swiggy_place_order` returns success — i.e. during the announce-then-call window or during retries — abort the call (don't retry further) and transition to CANCELLED.

### Coupon applied successfully but order placement fails

The coupon is held by Swiggy and may or may not be reusable. Don't worry about it — re-apply on retry, accept the new state Swiggy reports.

### Idempotency key from a previous session

Don't ever reuse an idempotency key across sessions. Generate fresh per session, at the start of PLACING. Store it on the session record.

### Multi-attempt placement with partial success

If your retries are returning ambiguous responses (some say "accepted", some say "duplicate"), assume the first "accepted" is the real one and call `swiggy_track_order` with that ID to confirm. The dedup behaviour of the idempotency key means you should converge on a single canonical order.

## Transition out

→ **COMPLETE** on Swiggy success (order ID captured)
→ **BROWSING** on restaurant-offline business error
→ **COLLECTING** on address / payment / stock business errors
→ **CANCELLED** on unrecoverable error, repeated network failures with leader giving up, or leader-initiated abort before success
