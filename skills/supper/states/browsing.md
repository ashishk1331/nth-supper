# BROWSING state

This is the first state the agent occupies after activation. The group is deciding *where* to order from, and you help them narrow.

## On entry (activation)

Before the discovery work begins, do these four things — in order:

1. **Check for an active session in this group.** If one already exists, do NOT start a new one. Reply with the existing reference and ask: *"There's already an active order — **#earlier-id**. Want me to cancel that first?"*
2. **Generate a session reference** in `#three-lowercase-words` format (e.g. `#swift-mango-lands`, `#brave-pepper-flies`). Make it memorable, never reuse a recent one.
3. **Pick a provisional party leader using this ladder:**
   - Group memory has a `default_leader` and they're in the chat → prefer them
   - Otherwise → the triggering user
4. **Load group memory** if available — usual restaurants, default address, recent ordering patterns, known dietary conflicts. Have these ready for the search step below.

Then send the opening message — acknowledgement + leader name + first discovery prompt. **Always name the leader so they can object within ~30 seconds.** Examples:

> "Started **#swift-mango-lands**. @alice on the bill, that work? What are we eating today, team?"

With memory (default leader from past sessions):

> "Started **#brave-pepper-flies**. @bob you usually run point — that or @alice taking it? Going Hyderabadi tonight, your usual is Paradise — that or somewhere new?"

With explicit cuisine:

> "Started **#calm-cinnamon-runs**. @alice paying — sound good? Pizza incoming. Ovenstory or Pizza Hut, or hunting?"

A guest objecting within ~30 seconds is a reassignment signal — see *Roles → Reassignment* in `SKILL.md`.

## Goal

Lock in exactly one restaurant — agreed by the group, or at least by the party leader — with its full menu fetched and cached locally for the rest of the session.

## Inputs to gather

You may need to elicit one or more of:

- **Cuisine** — Indian / Chinese / pizza / specific dish ("biryani", "ramen")
- **Constraints** — budget per head, max delivery time, dietary requirements
- **Old vs new** — use a usual from group memory, or try something fresh

You don't need all of these before searching — start with what you have and refine.

## Tool flow

```
swiggy_search_restaurants(query, cuisine, location, [filters])
  ↓ pick one based on group input + reactions
swiggy_get_menu(restaurantId)        ← exactly once, cache the result
```

`swiggy_get_dish_details` is for COLLECTING, not BROWSING. Don't call it yet.

## Actions

1. **Surface group history first** if memory is available:

   > "You usually order from Punjab Grill on Fridays — that, or somewhere new?"

   This is high-signal — groups have strong patterns and offering the usual saves three turns of search.

2. **Once you have a query, call `swiggy_search_restaurants`.** Pass cuisine, location, and any price/dietary filters the group mentioned.

3. **Show 3–5 options.** Numbered list, with name, rating, ETA, and price tier. Use buttons or rich blocks if your platform supports them.

   > Found these:
   > 1. **Punjab Grill** ⭐ 4.5 · 35 min · ₹₹
   > 2. **Karim's** ⭐ 4.3 · 40 min · ₹₹
   > 3. **Pind Balluchi** ⭐ 4.4 · 30 min · ₹₹₹
   >
   > Vibe?

4. **Watch for the group to converge.** Signals:
   - A user names a restaurant (typed reply)
   - A user reacts 🔥 / ❤️ to one option
   - The party leader picks

   If two restaurants get equal traction, ask the leader explicitly to break the tie. Don't pick for them.

5. **Once one is named, confirm and lock:**

   > "Going with **Punjab Grill**, ~35 min ETA. Locking it in."

6. **Call `swiggy_get_menu(punjabGrillId)` exactly once.** Cache the response in your local session memory. Every later question about dishes, prices, customisations, and availability uses this cache (plus targeted `swiggy_get_dish_details` for specific dishes).

7. **Announce the cart is open** as part of the transition message:

   > "Menu's up. What's everyone having?"

## Things NOT to do

- **Don't fetch menus for multiple candidates.** Wait until one restaurant is locked. Speculative menu fetches are wasteful and may hit rate limits.
- **Don't recommend more than 5 options at a time.** Choice fatigue kills group orders.
- **Don't lock the restaurant without group buy-in.** A leader's *"yes, that one"* or a clear majority of 🔥 reactions is the signal. Silence is not consent.
- **Don't volunteer the menu in the chat.** A 50-line menu dump is unreadable. Keep it cached and answer specific dish questions on demand.

## Edge cases

### No matches from `swiggy_search_restaurants`

Broaden the query (drop a filter, try a parent cuisine), ask the group to clarify, or fall back to suggestions from group memory.

### Group can't agree

Ask the party leader to decide. If they don't respond within ~3 minutes, suggest the highest-rated option and request a ✅:

> "@alice — going with Punjab Grill (top-rated). ✅ to confirm or name a different one."

### Restaurant has high minimum order value

Call it out now — don't let it ambush the group at COLLECTING time:

> "Heads up: ₹500 minimum at Punjab Grill. Should be fine for {n} people, just FYI."

### Restaurant is closed / unserviceable to the address

Treat as no-match: announce the issue, drop that option, surface alternatives.

> "Punjab Grill closed for the night. Karim's is open — that work?"

### Group memory suggests a restaurant that's no longer on Swiggy

The "usual" from memory may have delisted. If `swiggy_search_restaurants` for that name returns nothing, say so and offer alternatives:

> "Punjab Grill not on Swiggy right now. Pind Balluchi is similar (Punjabi, 4.4) — that work?"

## Transition out

→ **COLLECTING** when:
- Restaurant is locked AND
- `swiggy_get_menu` returned successfully AND
- You've announced the cart is open

→ Stay in BROWSING if the group is still debating, or if the chosen restaurant turned out to be unavailable and you need to surface alternatives.

→ **CANCELLED** if the group explicitly calls off the order in this phase ("nvm, we'll cook").
