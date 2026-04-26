# Sample session — Friday team lunch

A worked example showing the agent (`@supper`) running a complete group order from trigger to delivery. Ten visible chat exchanges spanning ~35 minutes. Each block has a brief *internal* note explaining what the agent should be doing under the hood.

## Setup

- **Group:** #lunch-team (a chat channel)
- **Members:** @alice, @bob, @carol
- **Time:** Friday 12:50 PM
- **Memory available:** yes — group has ordered ~6 times before, including 4 from Paradise Biryani on Fridays

---

### 1. Trigger

**[12:51] @alice:** *"@supper let's grab lunch, biryani vibes"*

**[12:51] @supper:**
> Started **#brave-pepper-flies**. @alice on the bill, that work? Going Hyderabadi for biryani — your usual is Paradise (you've ordered there 4 times). Try them again, or somewhere new?

> *Internal: skill activated, state BROWSING. Pre-flight: generated session id `#brave-pepper-flies`, ran the leader-assignment ladder (no `default_leader` in group memory → fell back to triggering user @alice as provisional leader), loaded group memory. Memory turned up Paradise as a strong Friday pattern, so surfaced it directly. Named @alice in the opening message so she or anyone else can object within ~30 seconds. No Swiggy tool calls yet — waiting for the group to converge on leader + restaurant.*

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

> *Internal: state BROWSING → COLLECTING. Called `swiggy_get_menu(paradise-id)` exactly once and cached the response. Pre-filled per-user suggestions from individual memory but did NOT auto-add anything — each member must say "yes" themselves. Acknowledged the high-min-order-value (₹500) up front so it doesn't ambush the group later.*

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
> Subtotal **₹960** (min ₹500 ✓). All set, @alice?

> *Internal: per-member carts populated. Total comfortably above min. Resolved each free-text item against the menu cache without needing `swiggy_get_dish_details` (no ambiguous variants in this case). Asked the party leader to close — closure is the leader's call, not unilateral.*

---

### 4. Leader closes; voting starts

**[12:56] @alice:** *"yep close it"*

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
> React ✅ to confirm. Closing in 10 min.

> *Internal: state COLLECTING → VOTING. Posted the voting summary as a tracked message — its message id is now bound to the session for reaction routing. Ran `swiggy_check_availability` on the cart in the background; all items in stock, no halt needed.*

---

### 5. Reactions roll in (silent)

**[12:57] reactions on the voting summary:**
- @alice: ✅
- @bob: ✅
- @carol: ✅

> *Internal: each ✅ on the tracked message marks `members[X].confirmed = true`. **No chat reply per reaction** — that would be noise. The agent waits until all three are in (or until timeout).*

---

### 6. All confirmed → placing window

**[12:58] @supper:**
> All set — placing in 10s. ❌ on this to abort.

> *Internal: all three members confirmed within 90 seconds — well before the 10-minute timeout. Posted the placing-in-10s message as a separate tracked message. Now waiting specifically for the party leader's ✅ on this new message (or anyone's ❌ to abort).*

---

### 7. Leader confirms; order placed

**[12:58] @alice:** ✅ (on the placing-in-10s message)

**[12:59] @supper:**
> Placing **#brave-pepper-flies** now…
>
> ✅ Placed at Paradise.
> Swiggy order: SW-9981234
> ETA: 31 minutes

> *Internal: state VOTING → PLACING → COMPLETE within ~30 seconds. Generated idempotency key `idem-9b3d-…` exactly once before calling `swiggy_place_order`. Call succeeded on the first attempt — no retry needed. Captured the Swiggy order id and ETA, then transitioned to COMPLETE with the success announcement.*

---

### 8. Tracking on demand

**[1:18] @bob:** *"where's it at"*

**[1:18] @supper:**
> Out for delivery, ~10 min away.

> *Internal: called `swiggy_track_order(SW-9981234)` because @bob asked. Did NOT broadcast every status update along the way — only on demand or at milestones.*

---

### 9. Delivery milestone

**[1:25] @supper:**
> Delivered for **#brave-pepper-flies**. ❤️ if you'd order from Paradise again.

> *Internal: tracked the order through to "delivered" status. Posted the delivery confirmation. The ❤️ reaction prompt feeds restaurant-affinity memory — group reactions on this message will be picked up by the async memory extraction.*

---

### 10. Reactions feed memory; session archived

**[1:26] reactions on the delivered message:**
- @alice: ❤️
- @bob: ❤️
- @carol: 🔥

> *Internal: state COMPLETE finalised. Session archived with status `complete`, full chat history, participant subtotals, swiggy order id. Memory extraction queued asynchronously. Expected updates from extraction:*
>
> *- Reinforce `(group)-[:USUALLY_ORDERS_FROM]->(Paradise)` — bump reinforceCount (now their 5th Friday)*
> *- Reinforce `(alice)-[:LIKES]->(chicken biryani)`, `(bob)-[:LIKES]->(paneer biryani)`, `(carol)-[:LIKES]->(veg biryani)`*
> *- New: `(carol)-[:LIKES]->(coke)` (first time on order)*
> *- Reinforce group fact `address.default = "Office, 3rd floor"`*
> *- Strong positive sentiment on Paradise → `affinity.score` bumped*

---

## What this example demonstrates

- **Full state progression:** BROWSING → COLLECTING → VOTING → PLACING → COMPLETE in under 35 minutes (most of which was waiting for delivery)
- **Memory leverage at BROWSING:** offered the group's usual restaurant proactively, saving 2–3 turns of search
- **Per-member memory at COLLECTING:** offered each user's past favourite as a *suggestion*, never auto-adding to a cart
- **Reactions as confirmation channel:** every ✅ on the voting summary was silent on the agent's side — no chat reply per reaction
- **Two distinct tracked messages:** the voting summary (for ✅/❌ on items) and the placing-in-10s message (for the leader's final placement ✅) — separate ids, separate intent
- **Idempotency:** key generated exactly once before `swiggy_place_order`, ready to be reused on retry (not needed here, but the discipline matters)
- **Tool economy:** `swiggy_get_menu` called *exactly once* in BROWSING; `swiggy_check_availability` once at the start of VOTING; `swiggy_track_order` only on user-initiated query and at the delivery milestone
- **Memory extraction at COMPLETE:** the agent did NOT manually call memory-write tools — extraction runs async. The agent only mentioned observations conversationally to the group when relevant (it didn't here, because the pattern was already known).
- **Concise messages:** every agent message fits on a screen. Cart summaries use bullets, not tables.

## What this example does NOT cover

For each of these, see the corresponding state file:

- **Member opts out mid-session** → `states/voting.md` and `states/collecting.md`
- **Restaurant unavailability** → `states/browsing.md` and `states/voting.md`
- **Cart total below minimum** → `states/collecting.md`
- **Voting timeout with partial confirmations** → `states/voting.md`
- **Network error during `swiggy_place_order`** → `states/placing.md`
- **Item out of stock at availability check** → `states/voting.md` and `states/collecting.md`
- **Reference to a previous session by `#human-id`** → `states/complete.md`
- **Order cancelled by Swiggy after placement** → `states/complete.md`
- **Group can't agree on a restaurant** → `states/browsing.md`
- **Coupon code application** → `states/placing.md`
