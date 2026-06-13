# chatterbox — Terse Mode With a Brain

**Origin:** derived from the open-source [caveman-skill](https://github.com/Shawnchee/caveman-skill) (MIT, © Shawnchee), then hardened through a 4-model review before adoption in a 100+ session production workflow. See `NOTICE` for attribution and the list of modifications.

---

## The problem

Agents spend a meaningful share of output tokens on ceremony: "I'd be happy to help!", play-by-play narration of tool calls, summaries of what they just did, offers to do more. On filler-heavy responses more than half the output can be ceremony; over a long session it compounds, because every verbose output becomes input on the next turn. Caveman-skill measured roughly 12-24% real session savings from cutting it.

## Why not just use caveman as-is

A 4-model review (two frontier models, two open-weight) of integrating caveman into a session-workflow system reached a unanimous verdict: the terse rules are sound, but an always-on terse mode has four failure modes for real teams — especially teams with non-technical members:

1. **It's binary.** Terse mode on a session that needed rich explanation wastes human time to save machine tokens — usually a bad trade.
2. **It has no audience awareness.** A non-technical founder reading "`userId` was number, cast added" has no idea what happened or whether it matters.
3. **It can strip the "why".** Decision rationale compressed away saves tokens today and costs debugging hours later.
4. **It invites false brevity.** An agent optimizing for short output is one step from skipping verification steps to look efficient.

ChatterBox keeps caveman's ten rules for execution output and adds four guards: an **adaptive trigger** (activated by measured token-burn trends, not permanently), an **audience gate** (human-facing explanation is never chatter), a **decision-context guard** (rationale and changelogs are never truncated), and a **no-false-brevity rule** (verification and destructive-action warnings are never skipped).

## The closed loop

ChatterBox is designed to pair with the `session-close` skill in this repo:

```
session-close economics footer measures token burn
        → burn above baseline? ChatterBox on next session
        → next footer proves the savings (or doesn't)
```

That feedback loop is the thing neither skill has alone: caveman saves tokens but can't tell you whether it did; an economics footer measures burn but can't do anything about it.

## Key insight

Token discipline is a measurement problem before it is a style problem. Turn on terse mode blindly and you trade human clarity for machine savings you never verify. Measure first, throttle second, verify third.
