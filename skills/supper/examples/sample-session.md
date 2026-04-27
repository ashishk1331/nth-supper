# Sample session — Friday team lunch

A worked example showing the agent (`@supper`) running a complete group order from trigger to delivery under the three-phase model. Each block has a brief *internal* note explaining what the agent is doing under the hood.

## Setup

- **Group:** #lunch-team (a chat channel)
- **Members:** @alice, @bob, @carol
- **Time:** Friday 12:50 PM
- **Memory available:** yes — group has ordered ~6 times before, including 4 from Paradise Biryani on Fridays

---

### 1. Trigger — PLANNING begins

**[12:51] @alice:** *"@supper let's grab lunch, biryani vibes"*

**[12:51] @supper:**
> Started **#brave-pepper-flies**. @alice on the bill, that work? Going Hyderabadi for biryani — your usual is Paradise (you've ordered there 4 times). Try them again, or somewhere new?

> *Internal: skill activated, phase PLANNING. Pre-flight: generated session id `#brave-pepper-flies`, ran the leader-assignment ladder (no `default_leader` → fell back to @alice), loaded group memory. Surfaced Paradise as a strong Friday pattern. Named @alice in the opening so reassignment can happen within ~30 seconds.*

---

### 2. Group converges on the restaurant

**[12:52] @bob:** *"Paradise works."*
**[12:52] @bob:** *"@carol you in?"*
**[12:52] @carol:** 👍

**[12:53] @supper:**
> Locking **Paradise Biryani** (~30 min, ₹500 min). Menu's up — what's everyone having?
>
> Quick tips from past orders:
> • @alice — usual chicken biryani + raita?
> • @bob — your last 3 here were paneer biryani, want that?
> • @carol — veg or non-veg today?

> *Internal: still in PLANNING. Called `get_restaurant_menu(paradise-id)` exactly once and cached. Pre-filled per-user suggestions from individual memory but did NOT auto-add — each member must say "yes" themselves. Acknowledged the high-min-order-value (₹500) up front.*

---

### 3. Members add their items

**[12:54] @alice:** *"yes usual + add a gulab jamun this time"*
**[12:54] @bob:** *"paneer biryani, no raita"*
**[12:55] @carol:** *"veg biryani and a coke"*

**[12:55] @supper:**
> Cart for **#brave-pepper-flies** at Paradise:
> • @alice — Chicken biryani, raita, gulab jamun ····· ₹420
> • @bob   — Paneer biryani ··························· ₹280
> • @carol — Veg biryani, Coke ························· ₹260
>
> Subtotal **₹960** (min ₹500 ✓)

> *Internal: per-member carts populated via `update_food_cart` for each addition. Resolved each free-text item against the menu cache without needing `search_menu` (no ambiguous variants). Agent does NOT prompt the leader to close — it watches for activity to die down.*

---

### 4. Auto-close — agent closes PLANNING itself

**[12:56] @supper:** *(after ~60s of no further activity)*
> Calling it — moving to vote.

> *Internal: all three members have items, total ₹960 well above the ₹500 min, chat has been quiet ~60 seconds. PLANNING auto-closes. The party leader does NOT close this phase under the three-phase model.*

---

### 5. VOTING begins

**[12:56] @supper:**
> **#brave-pepper-flies** — final cart at Paradise:
>
> • @alice — Chicken biryani, raita, gulab jamun ····· ₹420
> • @bob   — Paneer biryani ··························· ₹280
> • @carol — Veg biryani, Coke ························· ₹260
>
> Subtotal: ₹960 · Delivery: ₹40 · **Total: ₹1000**
> Delivery: @alice (party leader) — Office, 3rd floor
>
> React ✅ / 👍 to confirm, ❌ / 👎 to opt out. Or reply in chat. Closing in 10 min.

> *Internal: phase PLANNING → VOTING. Posted the voting summary as a tracked message — its message id is now bound to the session for reaction routing. No MCP tool calls in this phase.*

---

### 6. Reactions roll in (silent)

**[12:57] reactions on the voting summary:**
- @alice: ✅
- @bob: ✅
- @carol: 👍

> *Internal: each ✅/👍 marks `members[X].confirmed = true`. **No chat reply per reaction** — silent acknowledgement. All three responded in 60 seconds — well before the 10-minute timeout.*

---

### 7. ORDER — read back, leader confirms, place

**[12:58] @supper:**
> **#brave-pepper-flies** — placing this:
>
> • @alice — Chicken biryani, raita, gulab jamun ····· ₹420
> • @bob   — Paneer biryani ··························· ₹280
> • @carol — Veg biryani, Coke ························· ₹260
>
> Subtotal: ₹960 · Delivery: ₹40 · **Total: ₹1000**
> Delivery: Office, 3rd floor
>
> @alice — ✅ to place, ❌ to abort.

**[12:58] @alice:** ✅

**[12:59] @supper:**
> Placing **#brave-pepper-flies** now…
>
> ✅ Placed at Paradise.
> Order: SW-9981234 · ETA: 31 min
> Track: [link]

> *Internal: phase VOTING → ORDER. Called `get_food_cart` to read back. Generated idempotency key `idem-9b3d-…` exactly once before `place_food_order`. Single attempt succeeded. Captured order id + ETA, called `track_food_order` for the tracking link.*

---

### 8. Owe list — automatic

**[12:59] @supper:**
> **Owe list — #brave-pepper-flies** (paid by @alice):
> • @bob   — ₹293 (paneer biryani + delivery share)
> • @carol — ₹273 (veg biryani, coke + delivery share)
>
> Total collected: ₹566 of ₹1000. (₹40 delivery split equally; @alice's own subtotal not owed.)

> *Internal: owe list posted automatically right after the success message. Delivery split equally across all three paying members (₹13.33 each, rounded). @alice's own ₹420 subtotal is not owed back to herself.*

---

### 9. Tracking on demand

**[1:18] @bob:** *"where's it at"*

**[1:18] @supper:**
> Out for delivery, ~10 min away.

> *Internal: called `track_food_order(SW-9981234)` because @bob asked. No fixed-schedule polling — only on demand or at milestones.*

---

### 10. Delivered

**[1:25] @supper:**
> Delivered for **#brave-pepper-flies**. ❤️ if you'd order from Paradise again.

**[1:26] reactions:**
- @alice: ❤️
- @bob: ❤️
- @carol: 🔥

> *Internal: tracked the order through to "delivered". Session archived with `status: "complete"`, full chat, per-member subtotals, order id. Memory extraction queued asynchronously.*

---

## What this example demonstrates

- **Three-phase progression:** PLANNING → VOTING → ORDER, with the agent (not the leader) closing PLANNING automatically once activity went quiet.
- **Single explicit leader confirmation in ORDER** — one prompt, one ✅, then placement. No separate "placing in 10s" message.
- **Automatic owe list** posted right after the success message — the agent doesn't ask whether to share it.
- **Memory leverage at PLANNING:** offered the group's usual restaurant proactively.
- **Per-member memory in PLANNING:** offered each user's past favourite as a *suggestion*, never auto-adding to a cart.
- **Reactions as confirmation channel:** every ✅/👍 was silent on the agent's side.
- **Idempotency:** key generated exactly once before `place_food_order`.
- **Tool economy:** `get_restaurant_menu` called *exactly once* in PLANNING; `track_food_order` only on user-initiated query and at the delivery milestone.

## What this example does NOT cover

For each, see the relevant phase file:

- **Member opts out mid-session** → `states/voting.md` and `states/planning.md`
- **Restaurant unavailability** → `states/planning.md` and `states/voting.md`
- **Cart total below minimum** → `states/planning.md`
- **Voting timeout with partial confirmations** → `states/voting.md`
- **Network error during `place_food_order`** → `states/order.md`
- **Reference to a previous session by `#human-id`** → `states/order.md`
- **Order cancelled by Swiggy after placement** → `states/order.md`
- **Group can't agree on a restaurant** → `states/planning.md`
- **Leader reassignment** → `states/planning.md` (PLANNING-only)
