# VOTING phase

PLANNING just auto-closed. Members confirm their items or opt out. No new items, no restaurant changes.

## Goal

Every member has either confirmed (✅/👍) or opted out (❌/👎), OR the 10-minute timeout has elapsed with at least one confirmation.

## Tool flow

No Swiggy MCP calls in this phase. You're managing reactions and recomputing totals locally.

## Actions

1. **Post the voting summary message.** Remember its message ID — this is the *tracked message* reactions will land on:

   > **#swift-mango-lands** — final cart at Punjab Grill:
   >
   > • @alice — Paneer tikka, garlic naan ······· ₹290
   > • @bob   — Butter chicken, plain rice ······· ₹340
   > • @carol — Veg biryani ····················· ₹220
   >
   > Subtotal: ₹850 · Delivery: ₹40 · **Total: ₹890**
   > Delivery: @alice (party leader) — Office, 3rd floor
   >
   > React ✅ / 👍 to confirm, ❌ / 👎 to opt out. Or reply in chat. Closing in 10 min.

2. **Track confirmations and opt-outs.**
   - ✅ / 👍 from member X (reaction or message) → `members[X].confirmed = true`
   - ❌ / 👎 from member X → `members[X].optedOut = true`. Remove their items, **recompute the total**, post a brief update if the total changed materially.

3. **Acknowledge silently.** Do NOT reply in chat to every individual confirmation. The reaction is the acknowledgement.

4. **Ping at half-timeout** (5 minutes in). Mention only the unresponsive members:

   > "@bob @carol — last call. ✅ or ❌ on the cart above?"

5. **At full timeout** (10 minutes), close the vote. Exclude unresponsive members and announce who was dropped:

   > "Time's up. Going with @alice and @bob — @carol didn't respond, dropping their items."

6. **Once everyone has responded (or timeout reached with ≥1 confirmation), transition to ORDER.** No separate "placing in 10s" message — ORDER will read back the cart and ask the leader for the single explicit confirmation.

## Things NOT to do

- **Don't add items in VOTING.** If a member wants to add something, drop back to PLANNING and re-summarise.
- **Don't reply in chat to every reaction.** Silent acknowledgement only.
- **Don't ping the same member twice.** One ping at half-timeout is the limit.
- **Don't place the order in this phase.** ORDER does that.
- **Don't reassign the leader.** Reassignment is PLANNING-only.

## Edge cases

### Total drops below restaurant minimum after opt-outs

Drop back to PLANNING:

> "Down to ₹430 after @carol opted out — that's ₹70 below the min. Adding to the cart, anyone?"

### Voting timeout with zero confirmations

Cancel:

> "Nobody confirmed — calling off **#swift-mango-lands**. Restart whenever."

### Restaurant goes unavailable mid-vote

Drop back to PLANNING. The whole cart is now invalid:

> "Punjab Grill just went offline 🤦 — picking somewhere else."

### Members keep reacting and un-reacting

Take the latest state. If a member ✅ then ❌, treat them as opted out.

### Party leader opts out

The leader's ❌ in voting is unusual — they're abandoning their role, but reassignment is PLANNING-only in this model. Drop back to PLANNING and let them or someone else hand off the role there:

> "@alice opted out — heading back to planning so we can sort the leader."

## Transition out

→ **ORDER** when all active members have responded (or timeout with ≥1 confirmation) and total ≥ minimum.
→ **PLANNING** if total drops below minimum, restaurant becomes unavailable, or the leader opts out.
→ **CANCELLED** if zero confirmations at timeout.
