# supper-skills

> Agent Skills that let any AI agent run collaborative group food ordering sessions via Swiggy MCP — in Slack, Discord, Telegram, or any chat surface.

![supper banner](assets/banner.webp)

## What this is

`supper-skills` is a collection of behaviour skills, not an application. The first skill in this repo, `supper`, drops into any agent that supports the [Agent Skills open standard](https://agentskills.io/specification.md) and has access to the Swiggy MCP servers, and teaches the agent to coordinate a group food order from "let's order food" to "delivered".

The skill teaches the agent:

- A three-phase ordering flow — PLANNING → VOTING → ORDER
- Which Swiggy MCP tools to chain at each phase
- How to coordinate multiple humans building one shared cart
- How chat reactions become confirmations and votes
- How to assign a party leader for payment and delivery
- When to surface dietary preferences and group order history from memory
- How to reference orders with `#human-id` slugs like `#swift-mango-lands`
- How to handle edge cases — minimum order value, opt-outs, restaurant unavailability, voting timeouts, network errors

## Roles in a session

Three roles exist in every order:

- **Party leader** — one person, on the hook for the order. Pays, confirms the delivery address, and gives the **single explicit confirmation** in ORDER that triggers placement. Picked at session start (group memory's default leader if any, otherwise the user who triggered) and confirmed explicitly in the opening message. **Reassignable only during PLANNING.** Does NOT close PLANNING — the agent does.
- **Guests** — everyone else. Add and remove their own items freely during PLANNING, confirm (✅/👍) or opt out (❌/👎) during VOTING by reaction or message. No action required in ORDER. Can join at any point during PLANNING.
- **Agent** — owns the entire session flow: closes PLANNING automatically when activity goes quiet, nudges silent members once before excluding, manages voting and timeouts, recalculates totals on opt-outs, places the order after the leader's confirmation, and posts the owe list automatically after ORDER completes.

## Requirements

- An AI agent compatible with the Agent Skills standard (Claude Code, Goose, OpenCode, Kilocode, or any other Skills-aware agent)
- Access to the Swiggy MCP servers (food, instamart, dineout)
- A multi-user chat surface that supports `@mentions` and reactions

## Installation

### Recommended — `npx skills add`

The fastest way for any Skills-aware agent (Claude Code, Goose, OpenCode, Kilocode, etc.):

```bash
npx skills add ashishk1331/supper-skills/supper
```

This pulls `skills/supper/` from the GitHub repository and installs it into your agent's active skills directory.

## How to use it

Once installed, the skill auto-activates when someone in a group chat says something like:

- "let's order food"
- "start a group order"
- "order dinner for the team"
- "supper time"
- "what should we eat"

The agent will generate a session id (`#brave-pepper-flies`), open a cart, and run the rest of the flow. Members add items in plain English, react ✅ to confirm, and the party leader gives the single final go-ahead before the order is placed.

You don't need to teach members new commands — natural conversation works. Reactions on the agent's tracked messages (the voting summary and the leader's confirmation prompt in ORDER) are the primary confirmation channel.

## The ordering process

```
PLANNING → VOTING → ORDER
       ↘ CANCELLED
```

| Phase | What happens | Trigger to advance |
|---|---|---|
| **PLANNING** | Agent generates a session id, picks a party leader, and loads group memory. Group converges on one restaurant; menu fetched once via `get_restaurant_menu` and cached. Each member adds items in plain text via `update_food_cart`; `search_menu` resolves ambiguous dish names. Agent flags dietary conflicts silently and nudges silent members once before marking them opted-out. | Agent auto-closes when all active members have items or are opted-out, total ≥ minimum, and chat goes quiet (~60s) |
| **VOTING** | Agent posts a final per-person summary with subtotals and total. Members confirm (✅/👍) or opt out (❌/👎) by reaction or message. Opt-outs trigger total recalculation. Reactions are acknowledged silently. | All respond (or 10-minute timeout with ≥1 confirmation) |
| **ORDER** | Agent reads back the final cart via `get_food_cart`, prompts the leader for one explicit ✅. Idempotency key generated once. `place_food_order` is called; retries reuse the same key. Order id + ETA shared via `track_food_order`. **Owe list posted automatically.** Tracking continues on demand and at delivery milestones. | (terminal) |

Every phase has a detailed playbook in [`skills/supper/states/`](skills/supper/states/) — the agent reads the relevant file when it transitions in.

## A worked example

A complete Friday team lunch is in [`skills/supper/examples/sample-session.md`](skills/supper/examples/sample-session.md), showing every phase transition, every Swiggy tool call, the reaction-driven confirmations, and the auto-posted owe list.

## Skill structure

```
skills/supper/
├── SKILL.md                       Overview, invariants, when to activate
├── states/
│   ├── planning.md                Activation pre-flight, restaurant pick, menu fetch, per-member item collection, auto-close
│   ├── voting.md                  Summary message, reactions, opt-out recalculation, timeout handling
│   └── order.md                   Read-back, leader confirmation, idempotent placement, tracking, owe list
└── examples/
    └── sample-session.md          End-to-end group order
```

The skill follows the [Agent Skills specification](https://agentskills.io/specification.md): YAML frontmatter with `name` and `description`, body under 500 lines, references one level deep, progressive disclosure so reference files load only when needed.

## Customising

This skill is text. To adapt it:

- **Different food platform** — replace Swiggy MCP tool names in `SKILL.md` and each `states/*.md` with your platform's MCP tool names. The three-phase flow, coordination rules, and edge-case logic carry over unchanged.
- **Different reaction conventions** — the ✅/❌/🔥/😐 mapping in `SKILL.md` is illustrative; adjust to your group's culture.
- **Different timeouts** — the 10-minute voting window in `states/voting.md` is a default, not a hard rule.
- **Different ID format** — `#human-id` slugs are convention; any human-readable scheme works.

After editing, validate with `skills-ref validate ./skills/supper`.

## Architecture (for the curious)

This skill was extracted from a fuller project specification — a self-hostable bot that runs the same flow with persistent memory (Postgres + FalkorDB), session locks, and three-layer context-window compaction. The skill captures the *behavioural* layer of that design so any agent — even one without bespoke infrastructure — can run the conversation correctly.

If you're building the persistent backend, the original design lives in commit history under `ARCHITECTURE.md`.

## License

MIT — see [`LICENSE`](LICENSE).
