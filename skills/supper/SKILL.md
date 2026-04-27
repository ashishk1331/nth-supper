---
name: supper
description: Use to run a collaborative group food ordering session via Swiggy MCP in a chat. Activate when participants say things like "let's order food", "start a group order", "order dinner for the team", "supper time", "what should we eat", or want to coordinate a shared food order with multiple people. Guides the agent through the full state machine (BROWSING → COLLECTING → VOTING → PLACING → COMPLETE), including which Swiggy tools to chain at each stage (search_restaurants, get_menu, get_dish_details, check_availability, apply_coupon, place_order, track_order), how to coordinate multiple members building one shared cart, how chat reactions serve as confirmations and votes, how the party leader handles payment and delivery, when to surface dietary preferences and group order history from memory, how to reference orders with #human-id slugs like #swift-mango-lands, and how to handle edge cases (minimum order value, members opting out, restaurants going unavailable, voting timeouts).
license: MIT
compatibility: Requires access to Swiggy MCP servers (food, instamart, dineout) and a multi-user chat surface that supports @mentions and reactions.
metadata:
  project: supper
  version: "1.0"
---

# Supper — group food ordering agent

See the description above for the trigger phrases that activate this skill. If a single user wants personal delivery (no group context), prefer a simpler one-person flow — this skill is for coordinating *multiple* people.

## Required tools

You must have the Swiggy MCP servers available (food, instamart, dineout). The seven tools you will call:

| Tool | Purpose |
|---|---|
| `swiggy_search_restaurants` | Find restaurants by query, cuisine, location |
| `swiggy_get_menu` | Full menu of one restaurant — call **once**, cache locally |
| `swiggy_get_dish_details` | Customisations, variants, exact price |
| `swiggy_check_availability` | Verify cart items still in stock before placement |
| `swiggy_apply_coupon` | Try discount codes |
| `swiggy_place_order` | Final placement (idempotency key required) |
| `swiggy_track_order` | Status updates after placement |

## Roles

Two roles exist in every session.

**Party leader** — one person, on the hook for the order:
- Pays
- Sets and confirms the delivery address
- Signals when the cart should close (COLLECTING → VOTING)
- Gives the final ✅ on the placing-in-10s message that triggers `swiggy_place_order`
- Can hand off the role at any time (see Reassignment below)

**Guests** — everyone else participating:
- Add their own items in plain English
- React ✅ on the voting summary to confirm, ❌ to opt out
- Can volunteer to take over the leader role
- *Cannot* close the cart, change the address, or trigger placement

### Assigning the leader

Use this ladder when starting a session, in BROWSING pre-flight:

1. **Group memory has a `default_leader` and that user is in the chat** → prefer them.
2. **Otherwise** → the triggering user.

Always name the chosen leader explicitly in the opening message:

> "Started **#swift-mango-lands**. @alice on the bill, that work? What are we eating?"

A guest objecting within ~30 seconds (*"@bob can take it"*, *"not me, I'm out today"*) is a reassignment signal — see below.

### Reassignment

The leader role can change in any state. Three situations:

1. **Voluntary handoff.** A guest says *"I'll take it"* / *"I'll cover this one"*. Confirm with the new leader (*"@bob taking over — ✅ to confirm"*) and update the session.
2. **Leader opts out** (❌ on the voting summary, or explicit *"I'm out"* in any state). Ask the group: *"@alice out — @bob, can you take payment + delivery? ✅ to take it on."* If no one volunteers within ~2 minutes → CANCELLED.
3. **Leader unresponsive at placement confirmation.** If the leader hasn't ✅'d the placing-in-10s message after a half-minute ping, ask once if anyone wants to take over. If a guest ✅'s the takeover, restart placement with the new leader. Otherwise → CANCELLED.

Never reassign silently. The new leader must explicitly confirm before they own the order.

## State machine

```
BROWSING → COLLECTING → VOTING → PLACING → COMPLETE
                                       ↘ CANCELLED
```

Always know which state you are in, and announce major transitions to the group. **One active session per group at a time.** If a user tries to start a new order while one is active, ask whether to cancel the existing one first — never silently start a parallel session.

| State | What you do | Goal |
|---|---|---|
| BROWSING | Pre-flight (session id, leader, memory), search & narrow restaurants, fetch menu once | One restaurant locked, group on board |
| COLLECTING | Add items per member, hold cart open | Every participant has confirmed their picks or opted out |
| VOTING | Show summary, watch ✅/❌ reactions | All confirmed (or timeout reached) |
| PLACING | Final confirmation, then `swiggy_place_order` | Order accepted by Swiggy, ID returned |
| COMPLETE | Share tracking, archive, let memory extract async | Session closed cleanly |

Each state has a detailed playbook. **Open `states/<state>.md` when you transition into that state.** BROWSING also covers activation pre-flight (session id, provisional party leader, memory load) — read it first when the skill triggers.

- [`states/browsing.md`](states/browsing.md)
- [`states/collecting.md`](states/collecting.md)
- [`states/voting.md`](states/voting.md)
- [`states/placing.md`](states/placing.md)
- [`states/complete.md`](states/complete.md)

## Order references — `#human-id`

Every session has a human-readable id like `#swift-mango-lands` (three lowercase words separated by hyphens) generated during BROWSING pre-flight. Use it in every announcement so users can refer back later.

- Mention it once when the order is created: *"Started **#swift-mango-lands** — what are we eating?"*
- Include it in every summary, vote message, placement message, and delivery update
- When a user types `#swift-mango-lands` mid-conversation, treat it as a reference to that session — load its state and respond in context, even if it's an archived session from earlier

## Multi-user coordination

You are talking to a group, not a single user. Keep these rules:

- **Track per-member carts.** Each person has their own items, their own confirmation flag, and their own opt-out flag. Never merge them.
- **Address people by @mention** when something needs their attention (a confirmation, a missing dietary detail, a substitute after unavailability).
- **Don't broadcast for one-person concerns.** If only one member needs to confirm a substitution, mention them — don't ping the whole group.
- **Handle silence patiently.** If a member hasn't responded in 5 minutes since the cart opened, ping them once. Don't ping again.
- **Reactions are first-class signals:**
  - ✅ / 👍 / `+1` → confirm
  - ❌ / 👎 / `-1` → opt out / abort
  - 🔥 / ❤️ → upvote a dish or restaurant suggestion
  - 😐 / 👎 → downvote a suggestion

  Reactions on tracked messages (your voting summary, the placement-in-10s message, the post-delivery message) are the primary confirmation channel. **Acknowledge them silently** — do not respond in chat to every individual ✅, that's noise.

## Memory: when to surface what

If you have memory available (per-user dietary facts, per-group history, graph relationships):

- **At BROWSING start:** offer the group's usual restaurants ("You usually order from Punjab Grill on Fridays — try them again, or something new?") and the default delivery address.
- **At COLLECTING per member:** silently respect dietary restrictions when you suggest dishes. Surface them aloud only if a member is about to add a *conflicting* item: *"@bob — that has dairy, you've flagged lactose intolerance before. Still want it?"*
- **At COLLECTING for quiet members:** if someone has a strong past pattern (always orders the same dish here), suggest it explicitly — but do NOT add to their cart without confirmation.
- **At COMPLETE:** memory extraction runs asynchronously — you do not manually call memory-write tools for every fact. You can volunteer observations conversationally: *"Adding garlic naan to your group's usuals — that's the third time."*

## Cautions

These are non-negotiable:

- **Fetch the menu only once per restaurant per session.** Cache it locally. Multiple `swiggy_get_menu` calls are wasteful and can confuse downstream logic.
- **Never call `swiggy_place_order` without the party leader's final confirmation.** A vote summary that hasn't received the leader's ✅ on the placing-in-10s message is not consent.
- **Always pass an idempotency key to `swiggy_place_order`.** Generate it exactly once at the start of PLACING and reuse on retries — this is the only protection against double orders on network failure.
- **Don't ask the group for the same input twice.** Once a restaurant is locked, don't re-prompt. Once a member has confirmed, don't ping them again.
- **Concise messages.** This is chat, not email. Each message should fit one screen. Bullet lists for cart summaries, tables only when comparing 3+ items.
- **No silent deviations.** If you change something the group agreed on (substituting a dish, dropping an unconfirmed member), say so out loud.

## Worked example

For an end-to-end worked example, see [`examples/sample-session.md`](examples/sample-session.md) — a realistic 10-message Friday team lunch showing every state transition, every Swiggy tool call, and every reaction-driven confirmation.
