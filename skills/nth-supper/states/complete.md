# COMPLETE state

The order is placed. Now help the group track it, archive the session, and let memory extraction pick up learnings.

## Goal

Provide tracking on demand, post one delivery confirmation, archive the session record, and trigger asynchronous memory extraction.

## Tool flow

```
swiggy_track_order(orderId)    ← on demand and at delivery milestones
(archive session record)       ← move to long-term storage
(trigger memory extraction)    ← async, runs separately
```

## Actions

1. **Announce success with the essentials:**

   > ✅ **#swift-mango-lands** placed at Punjab Grill.
   > Swiggy order: SW-9981234
   > ETA: 32 minutes
   > Track: [link from `swiggy_track_order`]

   Include the order ID, restaurant, ETA, and a tracking link if Swiggy provides one. Keep it short — the group will ask follow-up questions if they want more.

2. **Set up tracking on demand.** Call `swiggy_track_order` when:
   - A user asks ("where's the food?", "ETA?")
   - You hit a milestone — once when the order goes "out for delivery", once when it's "delivered"

   Do NOT poll on a fixed schedule and broadcast every status. Status spam is annoying. Only *milestone* changes deserve a chat update.

3. **Post the out-for-delivery message** (at the relevant milestone):

   > "Out for delivery — ~10 minutes away."

4. **Post the delivered message:**

   > "Delivered for **#swift-mango-lands**. Hope it was good — react ❤️ if you'd order from Punjab Grill again."

   The reactions on this message feed restaurant-affinity memory.

5. **Archive the session.** All session data → long-term storage. Set:
   - `status: "complete"`
   - `placedAt: <timestamp>`
   - `swiggyOrderId: <captured>`
   - `chatHistory: <full transcript>`
   - `participants: <per-member subtotals + items>`

6. **Trigger memory extraction asynchronously.** This is a separate process — you don't run it inline. The extraction engine will:
   - Reinforce existing facts (member ordered the usual again → bump confidence)
   - Capture new facts (group ordered from a new restaurant successfully → add to usuals)
   - Update graph relationships: `(alice)-[:LIKES]->(paneer tikka)`, `(group)-[:USUALLY_ORDERS_FROM]->(restaurant)`, `(restaurant)-[:SERVES]->(cuisine)`
   - Reconcile contradictions (a member ordered something that contradicts a remembered preference → flag for review)

7. **Volunteer observations conversationally** if you noticed something worth surfacing:

   > "Adding Punjab Grill to your group's usuals — that's the third Friday in a row."

   These are messages to the *group*, not direct memory writes. The extraction engine handles the actual persistence.

## Things NOT to do

- **Don't keep the session "active" indefinitely.** After the delivered message, the session is archived. Future references to `#swift-mango-lands` should hit the archive, not a live session.
- **Don't broadcast tracking updates the group didn't ask for.** Status spam wastes attention.
- **Don't manually call memory-write tools for every fact you noticed.** The extraction engine handles persistence. You just observe and (optionally) volunteer in chat.
- **Don't ask for ratings.** The reaction-based "❤️ if you'd order again" is enough — explicit rating prompts feel like surveys.
- **Don't post the delivered message before Swiggy confirms delivery.** Trust `swiggy_track_order` for the milestone, not the ETA estimate.

## Edge cases

### User references the session after archival

A user types `#swift-mango-lands` later in the day or week. Treat it as a query against the archived session:

> "**#swift-mango-lands** at Punjab Grill — placed Friday 1pm, delivered 1:32, total ₹890.
> @alice: chicken biryani · @bob: paneer biryani · @carol: veg biryani."

If they're asking a follow-up like "was it good?", check if any reactions came in on the delivered message and report the sentiment.

### Order is cancelled by Swiggy after placement

Treat as a separate event: announce the cancellation, mark the archived session `status: "cancelled-by-restaurant"`, and offer to restart:

> "Punjab Grill cancelled **#swift-mango-lands** (out of an ingredient). Sorry. Reorder from Karim's?"

The cancellation flow is a transition out of COMPLETE back into a fresh BROWSING (with a new session id) — not a return to the same session.

### Refund or complaint

Swiggy handles refunds. Direct the leader to Swiggy support and capture the complaint into memory ("group had a bad experience with Punjab Grill — flag for future suggestions"):

> "For refunds you'll want Swiggy support directly. Noting the issue so we steer clear next time."

### Group orders again right after

A new "let's order" trigger after COMPLETE starts a fresh session with a *new* `#human-id`. Do not reuse the previous session reference.

### Some members never confirmed in voting

Their items weren't placed. Surface this once at COMPLETE if relevant:

> "@carol — you didn't confirm in time, so nothing for you in this order. Catch the next one."

### Network error during `swiggy_track_order`

Tracking is non-critical. If it fails, just say so:

> "Tracking is flaky right now — Swiggy will text/email when it's out for delivery."

Don't retry aggressively; tracking failures don't justify retrying the way placement does.

## Transition out

The session is now over. The skill is no longer active for the group. The next *"let's order"* trigger re-activates the skill and starts a fresh session with a brand-new `#human-id`, beginning at BROWSING.

If a user references this session later, you respond from the archive — but you do not re-enter COMPLETE. Each session is single-use.
