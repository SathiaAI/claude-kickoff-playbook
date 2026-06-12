# Agent Skills

Battle-tested skills from 100+ AI working sessions building a production consumer AI product — a solo founder working with an AI workforce.

These are not theoretical patterns. Each skill was extracted from a real failure, generalized, and used in production across hundreds of subsequent sessions.

---

## Skills

| Skill | Value prop | War story |
|-------|-----------|-----------|
| [council](council/) | Run a 3-4 model ensemble review before locking any significant decision. Unanimous = strong signal. Split = the finding. | Review discipline caught a leaked key in a runner script before it spread. Split verdicts exposed hidden assumptions in pricing and architecture. |
| [key-leak-triage](key-leak-triage/) | Triage before revoke. Find every consumer of a leaked key before pulling the plug, or risk turning a security incident into a production outage. | A local-only key rotated in 5 minutes instead of triggering a fire drill. The consumer-discovery step routinely contradicts what key labels claim. |
| [decision-ledger](decision-ledger/) | Append-only decision log that governs a rotating AI workforce across sessions and models. 130+ locked decisions, zero silent reversals. | A conflict check caught a new proposal that contradicted a locked pricing constraint — the model simply hadn't seen the decision because it wasn't in the ledger read at boot. |
| [session-close](session-close/) | End-of-session ritual: changelog, status snapshot, forbidden-string scan, audit trail check, optional session-starter handoff. | Three failure modes shaped this: archaeology (no snapshot), audit drift (auto-push racing manual commits), and a near-leak caught by the staged diff scan. |

---

## Install

Add this repo as a plugin source in Claude Code or a compatible Cowork environment:

```
/plugin marketplace add SathiaAI/claude-kickoff-playbook
```

Or install individual skills manually by copying the skill directory into your project's `.claude/skills/` folder and restarting your agent session.

---

## Why these exist

Every skill here was extracted from a real failure. The source failures are documented in each skill's `README.md`.

- **council/README.md** — why a solo founder runs a 4-model ensemble on every major call.
- **key-leak-triage/README.md** — why triage-before-revoke turns key leaks into routine rotations instead of production fires.
- **decision-ledger/README.md** — how a single markdown file governs a rotating cast of models across months.
- **session-close/README.md** — how 100 sessions with 5 different LLMs stay resumable by any model, cold.

The generalizing principle: AI-assisted projects fail not because the models are wrong, but because the scaffolding around the models is missing. These skills are scaffolding.

---

## License

These skills are MIT-licensed (see LICENSE in this directory). Fork, adapt, use. Attribution appreciated but not required.
