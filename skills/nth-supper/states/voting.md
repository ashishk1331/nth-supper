# VOTING state

The cart is closed. Members confirm via reactions.

## Goal

Every non-opted-out member has reacted ✅ on the voting summary message, OR the voting timeout (default 10 minutes) has elapsed with at least one ✅.

## Tool flow

```
swiggy_check_availability(cartItems)    ← run once at the start of VOTING
  ↓ all available
(post voting summary, watch reactions)
  ↓ all confirmed (or timeout)
(post placing-in-10s message)
  ↓ leader ✅ on that message
→ PLACING
```

## Actions

1. **Run `swiggy_check_availability`** with the full cart. If anything is unavailable, halt before posting the vote — handle the substitution first (see Edge cases).

2. **Post the voting summary message.** Remember its message ID — this is the *tracked message* that reactions will land on:

   > **#swift-mango-lands** — final cart at Punjab Grill:
   >
   > • @alice — Paneer tikka, garlic naan ······· ₹290
   > • @bob   — Butter chicken, plain rice ······· ₹340
   > • @carol — Veg biryani ····················· ₹220
   >
   > Subtotal: ₹850 · Delivery: ₹40 · **Total: ₹890**
   > Delivery: @alice (party leader) — Office, 3rd floor
   >
   > React ✅ to confirm your items, ❌ to opt out.
   > Closing in 10 minutes.

3. **Track reactions on this specific message:**
   - ✅ from member X → mark `members[X].confirmed = true`
   - ❌ from member X → mark `members[X].optedOut = true`, remove their items, recompute total

4. **Handle reactions silently.** Do NOT reply in chat to every individual ✅. The reaction itself is the acknowledgement. The next message you send is the all-confirmed announcement, not a per-confirmation reply.

5. **Ping at half-timeout** (5 minutes in, default). Mention only the unconfirmed members, not everyone:

   > "@bob @carol — last call. ✅ or ❌ on the cart above?"

6. **At full timeout** (10 minutes default), close the vote regardless. Exclude unconfirmed members and announce who was dropped:

   > "Time's up. Going with @alice and @bob — @carol didn't confirm, dropping their items."

   If at least one member confirmed, transition toward PLACING. If zero confirmed, cancel.

7. **Once everyone has confirmed (or timeout reached with ≥1 confirmation), post the placing-in-10s message:**

   > "All set — placing in 10s. ❌ on this message to abort."

   This is a separate tracked message. The party leader's ✅ on this message is the final signal. A ❌ from anyone aborts.

## Things NOT to do

- **Don't call `swiggy_place_order` from this state.** Placement happens in PLACING, after the leader's ✅ on the placing-in-10s message.
- **Don't add items in VOTING.** If a member wants to add something, you must drop back to COLLECTING and re-summarise — VOTING on a different cart than was confirmed is invalid.
- **Don't ignore an unavailability signal.** Placing an order containing an unavailable item will fail at Swiggy, and you'll have surprised the group.
- **Don't reply in chat to every reaction.** Silent acknowledgement only.
- **Don't ping the same member twice.** One ping at half-timeout is the limit.

## Edge cases

### Item unavailable at `swiggy_check_availability`

Halt. Before posting the vote summary, address it:

> "Heads up — paneer tikka just went out of stock. @alice — malai tikka instead (₹290, similar)?"

Once @alice picks, re-run `swiggy_check_availability` on the new cart, then post the vote summary.

If unavailability surfaces *after* the vote starts (rare), halt the vote, fix the cart, post a fresh summary, restart the vote.

### Member opts out and was the party leader

The leader's ❌ on the vote summary is special — they're not just opting out of items, they're vacating the leader role. Reassign before continuing:

> "@alice opted out — @bob, can you take payment + delivery? ✅ to take it on."

If no one volunteers within ~2 minutes → CANCELLED.

### Total drops below restaurant minimum after opt-outs

Drop back to COLLECTING:

> "Down to ₹430 after @carol opted out — that's ₹70 below the min. Adding to the cart, anyone?"

### Voting timeout with zero confirmations

Cancel:

> "Nobody confirmed — calling off **#swift-mango-lands**. Restart whenever."

Transition to CANCELLED.

### Restaurant goes unavailable mid-vote

Drop back to BROWSING. The whole cart is now invalid:

> "Punjab Grill just went offline 🤦 — picking somewhere else. Karim's, Pind Balluchi, or hunting?"

### Members keep reacting and un-reacting

Take the latest state. If a member ✅ then ❌, treat them as opted out. Don't pester for a tiebreaker.

### Leader takes too long on the placing-in-10s message

Wait ~30 seconds, then ping once:

> "@alice — ✅ on the message above to place, or ❌ to abort."

If still no response within another minute, offer reassignment before cancelling (see *Roles → Reassignment* in `SKILL.md`):

> "@alice still quiet — anyone want to take over payment + delivery? ✅ to take it on, or I'm cancelling in 60s."

If a guest ✅s the takeover, switch the leader, post a fresh placing-in-10s message under their name, and continue. If no one volunteers → CANCELLED: *"Nobody around to confirm — calling it off."*

## Transition out

→ **PLACING** when all confirmed (or timeout with ≥1 confirmation) AND the party leader has reacted ✅ on the placing-in-10s message.
→ **COLLECTING** if availability check failed and a substitute is needed, or if total drops below minimum after opt-outs.
→ **BROWSING** if the restaurant becomes unavailable.
→ **CANCELLED** if zero confirmations at timeout, leader vacates without replacement, or anyone reacts ❌ on the placing-in-10s message.
