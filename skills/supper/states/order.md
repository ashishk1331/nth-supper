# ORDER phase

The vote has closed. You read back the final order, get the leader's single explicit confirmation, place the order, share tracking, and post the owe list.

## Goal

Call `place_food_order` exactly once successfully, share the order id and ETA, and post the owe list breakdown — OR fail safely without ever double-charging.

## Tool flow

```
get_food_cart(sessionId)              ← read back the final cart
  ↓ leader's single explicit confirmation
(generate idempotency key — exactly once)
  ↓
place_food_order(cart, restaurant, address, idempotencyKey)
  ↓ success
track_food_order(orderId)             ← initial fetch + on milestones
  ↓
(post owe list automatically)
```

## Pre-flight checklist

Before calling `place_food_order`, verify:

- [ ] Vote has closed (all members responded, or timeout with ≥1 confirmation)
- [ ] **Idempotency key has been generated.** Generate now if not. Store it. Reuse on retries.
- [ ] Cart total ≥ restaurant minimum order value
- [ ] Delivery address is set and confirmed by the party leader
- [ ] Payment method is on file for the leader

## Actions

1. **Read back the final order** using `get_food_cart`. Post it as a single message and ask for the leader's one explicit confirmation:

   > **#swift-mango-lands** — placing this:
   >
   > • @alice — Paneer tikka, garlic naan ······· ₹290
   > • @bob   — Butter chicken, plain rice ······· ₹340
   >
   > Subtotal: ₹630 · Delivery: ₹40 · **Total: ₹670**
   > Delivery: Office, 3rd floor
   >
   > @alice — ✅ to place, ❌ to abort.

2. **Wait for the leader's reaction or message.**
   - ✅ / *"go"* / *"yes"* → proceed with placement.
   - ❌ / *"cancel"* → CANCELLED.
   - No response within ~60 seconds → ping once. If still silent after another 60 seconds, CANCELLED.

3. **Announce placement is happening:**

   > "Placing **#swift-mango-lands** now…"

4. **Generate the idempotency key once.** Store it. Reuse on retry.

5. **Call `place_food_order`** with the full final cart, restaurant id, delivery address, and the idempotency key.

6. **On success:**
   - Capture the order id and ETA.
   - Call `track_food_order(orderId)` for the initial status / tracking link.
   - Post the success message:

     > ✅ Placed at Punjab Grill.
     > Order: SW-9981234 · ETA: 32 min
     > Track: [link]

7. **Automatically post the owe list** right after the success message:

   > **Owe list — #swift-mango-lands** (paid by @alice):
   > • @bob   — ₹340 (butter chicken, rice)
   > • @carol — ₹220 (veg biryani)
   >
   > Total collected: ₹560 of ₹890 (₹40 delivery split equally; @alice's own subtotal not owed).

   Split delivery equally across all paying members unless the group has set a different convention. Don't ask whether to post — always post automatically.

8. **Track on demand.** Call `track_food_order` when:
   - A user asks ("ETA?", "where's the food?")
   - The order hits a milestone (out for delivery, delivered)

   Do NOT poll on a fixed schedule.

9. **Post the delivered message:**

   > "Delivered for **#swift-mango-lands**. Hope it was good — ❤️ if you'd order from Punjab Grill again."

   Reactions on this message feed restaurant-affinity memory.

10. **Archive the session.** Once delivered: `status: "complete"`, full chat, per-member subtotals, order id. Memory extraction runs asynchronously — you don't call memory-write tools yourself.

## On error

| Result | Action |
|---|---|
| Network error / timeout (no clear yes/no) | Retry with **same** idempotency key. Up to 3 attempts (1s, 2s, 4s backoff). |
| Restaurant offline | Announce, transition back to PLANNING. |
| Address unserviceable | Ask leader to update address; on update, retry. Otherwise CANCELLED. |
| Payment declined | Ask leader to retry; if they decline → CANCELLED. |
| Item out of stock | Transition back to PLANNING for substitution. |

After 3 failed network retries, ask the leader: *"Couldn't reach Swiggy after 3 tries — try again, or call it off?"*

## Things NOT to do

- **Never regenerate the idempotency key on retry.** That defeats the purpose.
- **Never call `place_food_order` more than once successfully.** If you got an order id, you are done.
- **Don't add or change items in this phase.** Drop back to PLANNING if needed.
- **Don't skip the owe list.** It's automatic.
- **Don't broadcast every tracking update.** Milestones and on-demand only.
- **Don't reassign the leader.** Reassignment is PLANNING-only. If the leader is unresponsive at confirmation, CANCELLED.
- **Don't announce success before the order id is in hand.**

## Edge cases

### Network error then a delayed success on retry

Idempotency dedup means Swiggy returns the same order id. Trust the key.

### Leader changes their mind during placement

Before `place_food_order` returns success → abort, transition to CANCELLED. After success → the order exists; point them at Swiggy support for cancellation.

### Order is cancelled by Swiggy after placement

Announce, mark archived session `cancelled-by-restaurant`, offer to restart with a new session id.

### User references the session later

Treat `#swift-mango-lands` as a query against the archived session. Report what was ordered, by whom, and the delivery outcome.

### Group orders again right after

Fresh trigger → new `#human-id` → new PLANNING. Never reuse a session reference.

## Transition out

The session ends here. Future *"let's order"* triggers start a new session in PLANNING with a brand-new `#human-id`.
