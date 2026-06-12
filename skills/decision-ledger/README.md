# decision-ledger — The Constitution for an AI Workforce

**Origin:** extracted from 100+ AI working sessions building a production consumer AI product — a solo founder working with an AI workforce.

---

## The problem

AI models are amnesiac. Every session starts cold. If you're running a multi-session, multi-model build — orchestrator, code-gen, adversarial reviewer, visual QA, spec-writer — none of them share memory. Without a durable, authoritative record of what was decided and why, each session is flying blind.

The failure mode is not dramatic. It's slow and quiet. One model re-opens a question you settled three weeks ago. Another builds a feature that contradicts a locked pricing constraint it didn't know about. A third proposes an architecture that violates a security boundary that exists for a reason nobody in the current context can reconstruct. None of this is malicious. It's just what happens when agents lack shared context.

## The failure that produced this

Pricing decisions nearly got contradicted twice. The first time, a new session proposed a feature that implied a pricing tier lower than the locked minimum — not because the model was wrong about the feature, but because it hadn't seen that the pricing was already settled. The second time, a council review for a different decision surfaced a proposal that would have required changing the commercial model — the conflict check caught it before anything was built.

The ledger started as a single markdown file with a handful of entries. It grew to cover 130+ locked decisions across the full arc of the product build: architectural splits, auth boundaries, commercial terms, regulatory posture, vendor choices, brand constraints. Not one of those locked decisions was silently reversed. When circumstances changed and an amendment was warranted, it was handled as a new sub-entry with an explicit paper trail.

The pattern that made it work is simple: every session reads the ledger before proposing anything. The ledger is not a changelog or a JIRA board. It is the constitution — the document that lets a rotating cast of models act consistently even though none of them have met each other.

## What the skill does

1. Defines a consistent entry format: ID, date, status, decision, alternatives rejected, traceability links.
2. Enforces append-only discipline: locked entries are never edited; amendments create new sub-entries.
3. Runs a structured conflict check before any entry is locked — catches contradictions with existing decisions on pricing, architecture, security, brand, and vendor axes.
4. Flags constitutional decisions (the fundamental constraints) and requires human owner named approval to amend them.
5. Manages ledger scale: a snapshot of LOCKED entries for fast boot reads, full ledger for audit.

## Key insight

The ledger's value compounds. At 10 decisions it's a reference. At 50 it's a constraint graph. At 100+ it's the institutional memory that makes a solo founder + AI workforce genuinely resilient across sessions, model changes, and long build timelines. The conflict check is what makes it active rather than passive — it doesn't just record history, it defends it.
