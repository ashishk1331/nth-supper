# PLANNING phase

The first phase. The group is deciding *where* to order from and *what* each person wants. You run discovery, hold the cart open, and close the phase yourself when activity dies down.

## On entry (activation)

Before discovery begins, do these in order:

1. **Check for an active session in this group.** If one exists, do NOT start a new one. Reply with the existing reference and ask: *"There's already an active order — **#earlier-id**. Cancel that first?"*
2. **Generate a session reference** in `#three-lowercase-words` format (e.g. `#swift-mango-lands`). Memorable, never reuse a recent one.
3. **Pick the party leader using this ladder:**
   - Group memory has a `default_leader` and they're in the chat → prefer them
   - Otherwise → the triggering user
4. **Load group memory** if available — usual restaurants, default address, recent patterns, dietary conflicts.

Then send the opening message — acknowledgement + leader name + first discovery prompt:

> "Started **#swift-mango-lands**. @alice on the bill, that work? What are we eating today, team?"

A guest objecting within ~30 seconds is a reassignment signal — see *Roles → Reassignment* in `SKILL.md`. **Reassignment is allowed only during PLANNING.**

## Goal

1. One restaurant locked, menu fetched and cached.
2. Every active member has at least one item in their cart, OR has been marked opted-out after one nudge.

You — the agent — close PLANNING automatically when both conditions hold and chat activity has gone quiet for ~60 seconds. The party leader does *not* close this phase.

## Tool flow

```
search_restaurants(query, cuisine, location, [filters])
  ↓ pick one based on group input + reactions
get_restaurant_menu(restaurantId)        ← exactly once, cache locally
  ↓ as members add items
search_menu(restaurantId, query)         ← resolve free-text dish names
update_food_cart(memberId, item, qty)    ← per member, idempotent
```

## Restaurant discovery

1. **Surface group history first** if memory is available:

   > "You usually order from Punjab Grill on Fridays — that, or somewhere new?"

2. **Once you have a query, call `search_restaurants`.** Pass cuisine, location, and any filters mentioned.

3. **Show 3–5 options** — numbered list, with name, rating, ETA, price tier.

4. **Watch for convergence.** A user names a restaurant, reacts 🔥/❤️ to one option, or the leader picks. If two get equal traction, ask the leader to break the tie.

5. **Lock and announce.** Once one is named, confirm and call `get_restaurant_menu(restaurantId)` exactly once. Cache the response.

   > "Going with **Punjab Grill**, ~35 min ETA. Menu's up — what's everyone having?"

## Item collection

1. **Track per-member carts.** Each user has their own `MemberCart`:
   - `items: CartItem[]`
   - `optedOut: boolean`

   Never merge into one anonymous cart.

2. **Resolve free-text orders against the menu cache.** When a user says *"I'll have the paneer tikka"*:
   - Exactly one match → call `update_food_cart` to add it, acknowledge with a ✅ reaction.
   - Multiple matches → call `search_menu(restaurantId, "biryani")` and ask: *"@alice — small or large biryani? Veg or chicken?"*
   - No match → say so: *"@alice — don't see paneer tikka on this menu. Got an alt?"*

3. **Surface dietary conflicts silently from memory.** If a member is about to add an item that contradicts a known restriction, inform them — don't block.

4. **Post a running cart summary every 5–10 user actions:**

   > Cart for **#swift-mango-lands** at Punjab Grill:
   > • @alice — Paneer tikka, garlic naan ······ ₹290
   > • @bob   — Butter chicken, plain rice ····· ₹340
   > • @carol — *(waiting)*
   >
   > Subtotal **₹630** (min ₹500 ✓)

5. **Cross-add coordination.** If @alice says *"add a coke for @bob too"*, never silently add to someone else's cart. Ask @bob to confirm.

## Auto-close

PLANNING ends when **all** of:

- Every active member has either ≥1 item OR is marked `optedOut`
- Cart total ≥ restaurant minimum
- Chat has been quiet for ~60 seconds

When you're about to auto-close, post the transition itself — don't ask permission:

> "Calling it — moving to vote."

## Silent members

If a member hasn't responded ~3 minutes after the cart opened, ping them **once**:

> "@carol — joining or skipping?"

If they don't respond within another ~2 minutes, mark them `optedOut` and announce:

> "Marking @carol out — they can jump back in if they show up."

One nudge, then exclude. Don't ping again.

## Things NOT to do

- **Don't fetch menus for multiple candidates.** One restaurant, one menu, one cache.
- **Don't recommend more than 5 restaurants at a time.**
- **Don't lock the restaurant without group buy-in.** A leader's *"yes"* or a clear majority of 🔥 reactions is the signal.
- **Don't dump the full menu in chat.** Answer specific dish questions on demand.
- **Don't auto-confirm a member's items for them.**
- **Don't re-fetch the menu** after the initial `get_restaurant_menu`.
- **Don't ask the leader to close.** That's your job in this model.
- **Don't include opted-out members in the running total.**

## Edge cases

### No matches from `search_restaurants`

Broaden the query, ask the group to clarify, or fall back to memory.

### Group can't agree on a restaurant

Ask the leader to decide. If they don't respond within ~3 minutes, suggest the highest-rated option and request a ✅.

### Restaurant has high minimum order value

Call it out now:

> "Heads up: ₹500 minimum at Punjab Grill. Should be fine for {n} people."

### Total below restaurant minimum at auto-close time

Don't close. Flag and suggest add-ons:

> "₹50 short of the ₹500 min. Add a side? Garlic naan ₹60, lassi ₹80."

### Restaurant goes unavailable

Drop the option, surface alternatives, restart restaurant selection in this same phase.

### Leader reassignment

A guest says *"I'll take it"* / *"@bob can take it"*. Confirm with the new leader (*"@bob taking over — ✅ to confirm"*) and update the session. Allowed only here, in PLANNING.

## Transition out

→ **VOTING** when all active members have items or are opted-out, total ≥ minimum, and chat has been quiet ~60 seconds.

→ **CANCELLED** if the group explicitly calls off the order ("nvm, we'll cook").
