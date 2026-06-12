---
name: decision-ledger
description: Manage an append-only decision log for AI-assisted projects, preventing silent reversals across sessions and models. Handles writing new entries, locking decisions, checking for conflicts with existing locked entries, and reading the ledger at session boot. Use when the user says "log this decision", "lock this decision", "add to the decision log", "check if this contradicts a past decision", "decision ledger entry", "D-NN log", "mark this as locked", "run the conflict check", "what decisions have we made", "read the ledger before we start", or "amend a previous decision".
---

# Decision Ledger

An AI workforce is amnesiac by default. Without a durable, append-only record of what was decided and why, models re-litigate settled questions, silently violate past constraints, and contradict each other across sessions. The ledger is the constitution.

## When to use

- At the start of any session involving a project with existing decisions — read the ledger before proposing anything.
- When a decision is being made that is non-trivial to reverse.
- When a human owner explicitly locks a decision after deliberation.
- Before proposing something new — run a conflict check against locked entries.
- When a past decision needs amending — create a new sub-entry, never edit the original.

## When NOT to use

- Routine implementation details (which function to name, which variable, code style).
- Decisions that will be revisited every sprint by design (sprint goals, backlog priorities).
- Anything that does not bind future sessions or future models to any constraint.

---

## Entry format

Every decision entry follows this structure. Keep entries concise — the ledger must be scannable.

```markdown
## D-NNN — [Short Title]

**Date:** YYYY-MM-DD
**Status:** DRAFT | LOCKED | AMENDED (see D-NNN.a)
**Owner:** [Name or role who can amend this — "Human Owner" for constitutional decisions]

**Decision:**
[One paragraph. What was decided. State it in the positive ("We will X") not the negative ("We won't not-X").]

**Alternatives rejected:**
- [Option A] — [one-sentence reason rejected]
- [Option B] — [one-sentence reason rejected]

**Traceability:**
- Implementing commit: [hash or "pending"]
- Review artifact: [council RAW file path, or "n/a"]
- Tracker issue: [issue ID or "n/a"]
```

**Amendment format** (new sub-entry, never edit the parent):

```markdown
## D-NNN.a — [Short Title of Amendment]

**Date:** YYYY-MM-DD
**Status:** LOCKED
**Amends:** D-NNN
**Owner:** [Human Owner — constitutional amendments require named approval]

**Amendment:**
[What changed from D-NNN and why. Reference D-NNN explicitly.]

**Traceability:**
- Implementing commit: [hash]
- Review artifact: [council RAW file or "n/a"]
```

---

## Step-by-step procedures

### A. Boot procedure — read the ledger

At the start of any session on a project that has a decision ledger:

1. Read the ledger file (or the snapshot of LOCKED entries if the ledger is large).
2. Extract the numbered list of all LOCKED decision titles and their one-sentence summaries.
3. Hold this set in context before generating any proposal.
4. If no ledger exists yet, note this and offer to initialize one.

Do not proceed with any substantive proposal until the ledger has been read.

### B. Writing a new entry

1. Assign the next sequential ID (D-001, D-002, ...). Check the ledger to confirm the ID is not already taken.
2. Set status to DRAFT.
3. Fill in all fields: date, owner, decision, alternatives rejected, traceability (use "pending" for commit hash if not yet implemented).
4. Append to the end of the ledger file. Do not insert inline or reorder existing entries.
5. Confirm the append with the user before saving.

### C. Conflict check — run before locking

Before marking any DRAFT entry as LOCKED, run a conflict check:

1. List all currently LOCKED decisions.
2. For each locked decision, ask: does this new decision contradict, supersede, or require amending a locked entry?
3. Check these specific axes:
   - **Pricing/commercial terms** — new pricing contradicts locked pricing tiers?
   - **Architecture** — new service split contradicts locked architectural split?
   - **Security/permissions** — new auth model contradicts locked auth boundary?
   - **Brand/regulatory** — new marketing claim contradicts locked brand constraints?
   - **Vendor lock-in** — new dependency contradicts a locked vendor-neutral stance?

If a conflict is found:
- Do NOT silently proceed.
- Surface the conflict explicitly: "This proposal conflicts with D-042 (locked YYYY-MM-DD): [what the conflict is]. Options: (a) amend D-042 first, (b) modify this proposal to be consistent, (c) escalate to human owner."
- If the conflicting locked entry is a constitutional decision, human owner approval is required before any path forward.

**Example:** A proposal adds a new pricing tier at $19/month. The conflict check finds D-031, which locks the Pro plan at $29/month as the floor with no tier below it. Conflict surfaced: "This contradicts D-031 (pricing locked). Proposing a $19 tier would require amending D-031 with human owner approval."

### D. Locking a decision

1. Confirm the conflict check is complete (step C above).
2. If the decision involved a council review, confirm the RAW transcript is saved and the traceability link is filled in.
3. Change status from DRAFT to LOCKED.
4. If this is a constitutional decision (one of the ~dozen that define the product's fundamental constraints), add a `CONSTITUTIONAL` flag and require the human owner to explicitly say "I approve locking D-NNN" — do not auto-lock on their behalf.
5. Announce the lock: "D-NNN is now LOCKED. Future sessions must not contradict this without amending it."

### E. Amending a locked decision

1. Verify the amendment is warranted (new information, not re-litigation of settled arguments).
2. If the parent entry is CONSTITUTIONAL, require explicit human owner named approval before proceeding.
3. Write a new sub-entry D-NNN.a (or .b if .a exists). Never edit the parent entry.
4. Run a conflict check on the amendment itself against all other locked entries.
5. Lock the sub-entry once approved.
6. Update the parent entry's status to `AMENDED (see D-NNN.a)` — this is the only permitted edit to a locked entry.

### F. Managing ledger size

When the ledger grows large enough that reading it all at boot is impractical (rough guide: over 200 entries or 50K tokens):

1. Maintain a separate LOCKED-SNAPSHOT.md containing only the title, one-sentence summary, and status of all LOCKED entries.
2. The full ledger retains every entry verbatim for audit.
3. Boot procedure reads the snapshot; drill-down reads the full entry as needed.
4. Regenerate the snapshot whenever new entries are locked.

---

## Constitutional decisions

Mark a decision CONSTITUTIONAL if it meets any of these criteria:

- Defines who the product is for (target market, use case)
- Defines what the product is NOT (exclusions, regulatory posture)
- Defines the commercial model (pricing structure, licensing)
- Defines a security invariant (key custody, encryption model, auth boundary)
- Defines a vendor relationship with significant switching cost

Constitutional decisions require human owner named approval to amend. The ledger should have at most a dozen of these — if everything is constitutional, nothing is.

---

## Failure modes

| Failure | What happens | Fix |
|---------|-------------|-----|
| New model re-litigates a locked decision | Wasted session time, potential contradictory build | Boot procedure reads ledger first; model must see locked decisions before proposing |
| Amendment edits original entry | Audit trail broken; can't see what was originally decided | Only sub-entries allowed; parent entry is append-only |
| Conflict check skipped | New decision silently contradicts a locked one | Conflict check is mandatory before locking, not optional |
| Ledger too large to read at boot | Boot procedure skipped; model acts without context | Maintain LOCKED-SNAPSHOT.md; regenerate on each new lock |
| Constitutional decision amended without owner approval | Fundamental constraint changed without accountability | CONSTITUTIONAL flag + named-approval gate |

---

## What this skill does NOT do

- Does not run the council review — use the `council` skill for multi-model validation before locking.
- Does not enforce decisions in code — it records them; enforcement is the build's responsibility.
- Does not resolve conflicts between locked decisions — it surfaces them for the human owner to decide.
