# council — Multi-Model Decision Validation

**Origin:** extracted from 100+ AI working sessions building a production consumer AI product — a solo founder working with an AI workforce.

---

## The problem

When you're building with AI as your primary workforce, you face a subtle failure mode: the model you're working with is also the model reviewing its own work. It will find flaws, but it will systematically miss the ones it's blind to. You're not getting a second opinion — you're getting a reflection.

The obvious fix is to ask a different model. The harder question is: how do you do that reliably, cheaply, and in a way that produces a durable audit trail?

## The failure that produced this

Early in the build, several major architectural choices were locked after a single-model review. They held up — but only because they were lucky. The pattern that revealed the gap was a pricing decision: one model recommended a pricing tier structure with confidence. The same brief sent to three other models came back with a 2-2 split. The split wasn't a problem — it was the finding. The disagreement surface was exactly where the hidden assumption lived.

Two months later, the discipline around the council caught something more concrete: a runner script used to orchestrate the council calls itself contained the gateway key in plain text. Before that script's history spread any further, review caught it. The cost of missing that would have been a live key in version history.

The council has run on every major decision since — architecture, security boundaries, pricing, and vendor choices. The unanimous verdicts build confidence fast. The split verdicts have consistently been the most valuable: they surface the exact axis where a decision is fragile, which is precisely what you need to know before you lock it.

## What the skill does

1. Structures a decision brief with options, constraints, and cost-of-being-wrong.
2. Mints a scoped virtual key (dollar cap, TTL) so the master key never appears in chat.
3. Smoke-tests model aliases before sending the real brief — slugs deprecate without warning.
4. Sends the identical brief to 3-4 diverse frontier models.
5. Synthesizes structured verdicts: unanimous means strong signal; split means surface the disagreement.
6. Writes a verbatim RAW transcript for audit, linked from the decision log.

## Who this is for

Any practitioner running AI-assisted work where decisions have real consequences — solo builders, small teams, or anyone who wants to know their architecture held up against more than one opinion before it gets expensive to change.

## Key insight

The council is not a voting machine. The split is the product. When models disagree, you've found the fragile assumption. When they agree, you've got genuine signal. Either way, you know more than you did before.
