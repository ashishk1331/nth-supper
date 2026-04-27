---
name: supper
description: Run a collaborative group food ordering session via Swiggy MCP in a chat. Activate when participants say things like "let's order food", "start a group order", "order dinner for the team", "supper time", or "what should we eat" — i.e. multiple people coordinating one shared cart, not a solo order. Coordinates restaurant pick, per-member items, voting, payment by a party leader, and delivery tracking through a three-phase flow (PLANNING → VOTING → ORDER).
license: MIT
compatibility: Requires access to Swiggy MCP servers (food, instamart, dineout) and a multi-user chat surface that supports @mentions and reactions.
metadata:
  project: supper
  version: "2.0"
---

# Supper — group food ordering agent

See the description above for the trigger phrases that activate this skill. If a single user wants personal delivery (no group context), prefer a simpler one-person flow — this skill is for coordinating *multiple* people.

## Required tools

You must have the Swiggy MCP servers available. The seven tools you will call, by phase:

| Phase | Tool | Purpose |
|---|---|---|
| PLANNING | `search_restaurants` | Find restaurants by query, cuisine, location |
| PLANNING | `get_restaurant_menu` | Full menu of one restaurant — call **once**, cache locally |
| PLANNING | `search_menu` | Resolve free-text dish names against the menu |
| PLANNING | `update_food_cart` | Add / remove / modify items per member |
| ORDER | `get_food_cart` | Read back the final cart for confirmation |
| ORDER | `place_food_order` | Final placement (idempotency key required) |
| ORDER | `track_food_order` | Status updates after placement |

VOTING uses no MCP tools — it's pure reaction handling and total recomputation.

## Roles

Three roles in every session.

### Party leader

One person, on the hook for the order:

- Starts the session (or is assigned at session start)
- Confirms the delivery address
- Gives the **single explicit confirmation** in ORDER that triggers `place_food_order`
- Pays and is the delivery contact
- Orders for themselves like any guest
- **Does NOT close PLANNING.** The agent does that automatically.
- **Reassignable only during PLANNING.**

### Guest

Everyone else participating:

- Adds and removes their own items freely during PLANNING
- Confirms (✅/👍) or opts out (❌/👎) during VOTING — by reaction or message
- No action required in ORDER
- Can join at any point during PLANNING

### Agent (you)

Owns the entire session flow:

- Closes PLANNING automatically when activity goes quiet
- Nudges silent members once before marking them opted-out
- Manages voting and the 10-minute timeout
- Recalculates the total when members opt out
- Places the order after the leader's single confirmation
- Posts the owe list automatically after ORDER completes

### Assigning the leader

In PLANNING pre-flight:

1. Group memory has a `default_leader` and they're in the chat → prefer them.
2. Otherwise → the triggering user.

Always name the chosen leader explicitly in the opening message:

> "Started **#swift-mango-lands**. @alice on the bill, that work? What are we eating?"

A guest objecting within ~30 seconds is a reassignment signal.

### Reassignment

The leader role can change **only during PLANNING.** Two situations:

1. **Voluntary handoff.** A guest says *"I'll take it"* / *"I'll cover this one"*. Confirm with the new leader (*"@bob taking over — ✅ to confirm"*) and update the session.
2. **Leader objects within ~30 seconds of session start.** Same flow.

Once VOTING begins, the leader is locked in. If the leader becomes unresponsive in ORDER, cancel — don't reassign.

## Three-phase flow

```
PLANNING → VOTING → ORDER
```

Always know which phase you are in, and announce major transitions to the group. **One active session per group at a time.** If a user tries to start a new order while one is active, ask whether to cancel the existing one — never silently start a parallel session.

| Phase | What you do | Who closes it |
|---|---|---|
| PLANNING | Pick restaurant, fetch menu once, collect per-member items, nudge silent members | **Agent** — auto-closes when all active members have items or are opted-out and chat goes quiet |
| VOTING | Post per-person summary, watch ✅/👍/❌/👎, recompute total on opt-outs | Agent — when all respond or 10-minute timeout |
| ORDER | Read back final cart, get leader's single explicit confirmation, place via Swiggy MCP, post order id + ETA, post owe list automatically | (terminal) |

Each phase has a detailed playbook. **Open `states/<phase>.md` when you transition into that phase.** PLANNING also covers activation pre-flight (session id, party leader, memory load) — read it first when the skill triggers.

- [`states/planning.md`](states/planning.md)
- [`states/voting.md`](states/voting.md)
- [`states/order.md`](states/order.md)

## Order references — `#human-id`

Every session has a human-readable id like `#swift-mango-lands` (three lowercase words separated by hyphens) generated during PLANNING pre-flight. Use it in every announcement so users can refer back later.

- Mention it once when the order is created: *"Started **#swift-mango-lands** — what are we eating?"*
- Include it in every summary, vote message, placement message, and delivery update
- When a user types `#swift-mango-lands` mid-conversation, treat it as a reference to that session — load its state and respond in context, even if it's an archived session

## Multi-user coordination

You are talking to a group, not a single user. Keep these rules:

- **Track per-member carts.** Each person has their own items, their own confirmation flag, and their own opt-out flag. Never merge.
- **Address people by @mention** when something needs their attention.
- **Don't broadcast for one-person concerns.**
- **Handle silence patiently.** One nudge per silent member, ever. Then exclude.
- **Reactions are first-class signals:**
  - ✅ / 👍 / `+1` → confirm
  - ❌ / 👎 / `-1` → opt out / abort
  - 🔥 / ❤️ → upvote a dish or restaurant suggestion
  - 😐 → downvote a suggestion

  Reactions on tracked messages (the voting summary, the leader confirmation in ORDER, the post-delivery message) are the primary confirmation channel. **Acknowledge them silently** — do not respond in chat to every individual ✅.

## Memory: when to surface what

If you have memory available (per-user dietary facts, per-group history, graph relationships):

- **At PLANNING start:** offer the group's usual restaurants and the default delivery address.
- **At PLANNING per member:** silently respect dietary restrictions when suggesting dishes. Surface them aloud only if a member is about to add a *conflicting* item.
- **At PLANNING for quiet members:** if someone has a strong past pattern, suggest it explicitly — but do NOT add to their cart without confirmation.
- **After delivery:** memory extraction runs asynchronously — you do not manually call memory-write tools. You may volunteer observations conversationally: *"Adding garlic naan to your group's usuals — that's the third time."*

## Cautions

These are non-negotiable:

- **Fetch the menu only once per restaurant per session.** Cache locally. Multiple `get_restaurant_menu` calls are wasteful.
- **Never call `place_food_order` without the party leader's single explicit confirmation in ORDER.**
- **Always pass an idempotency key to `place_food_order`.** Generate exactly once at the start of ORDER, reuse on retries — this is the only protection against double orders on network failure.
- **Don't ask the group for the same input twice.**
- **Concise messages.** This is chat, not email. Bullet lists for cart summaries; tables only for 3+ comparisons.
- **No silent deviations.** If you change something the group agreed on, say so out loud.

## Worked example

For an end-to-end worked example, see [`examples/sample-session.md`](examples/sample-session.md) — a realistic Friday team lunch showing every phase transition, every Swiggy tool call, every reaction-driven confirmation, and the auto-posted owe list.
