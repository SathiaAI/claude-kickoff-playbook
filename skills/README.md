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
| [session-close](session-close/) | End-of-session ritual: changelog + session-economics footer (tokens/cost/contract score), status snapshot, forbidden-string scan, audit trail check, next-session contract with binary acceptance criteria and a scope gate, optional session-starter handoff. | Three failure modes shaped the original: archaeology (no snapshot), audit drift (auto-push racing manual commits), and a near-leak caught by the staged diff scan. The economics + contract loop was added after a 4-model review: 4/4 accept-with-changes. |
| [chatterbox](chatterbox/) | Adaptive terse-output mode: cuts filler/narration/postamble (~12-24% output-token savings) with four guards — adaptive trigger, audience gate, decision-context guard, no false brevity. Derived from [caveman-skill](https://github.com/Shawnchee/caveman-skill) (MIT). | A 4-model review of always-on terse mode found four failure modes (binary, audience-blind, rationale-stripping, false brevity); ChatterBox is caveman rules plus the four guards, with the session-close economics footer as the trigger. |
| [deps-pr-triage](deps-pr-triage/) | Drain the Dependabot/Renovate queue without merging blindly. AI reviewer version claims are near-certain false positives — verify against the registry API. Major bumps always go to a dedicated session. | An AI review gate flagged a dependency version as nonexistent. The registry confirmed the version was real. npm 10's `--omit=dev` also proved unreliable; the lockfile-based filter replaced it. |
| [sequential-merge-discipline](sequential-merge-discipline/) | Land multiple PRs through an automated gate without silently losing content. Stacked simultaneous submissions can leave a PR merging into an orphaned base with the content gone. | Two PRs submitted simultaneously through an auto-merge gate: the second silently lost its content and had to be re-landed as a fresh PR. A separate incident: inferring the source branch from local state rather than the API caused a force-push. |
| [public-repo-scrub](public-repo-scrub/) | Move content from private to public without leaking internals. Grep the entire tree, not just your files. After history rewrites, stale tags re-expose scrubbed commits. | An independent full-tree grep caught internal references in template files that drafting had missed. After a history rewrite to remove a secret, all existing tags still pointed at the pre-rewrite commits. |
| [env-var-verification](env-var-verification/) | Write runbooks with variable names that match the deployed code. Documents are not ground truth — the code at the deployed SHA is. | A runbook quoted a variable name from a planning document. The name had been renamed in code before deploy. The misconfiguration caused ~30 minutes of silent failure in production identity resolution. |

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
- **deps-pr-triage/README.md** — why AI reviewer version claims are false positives and why npm 10 audit needs a lockfile filter.
- **sequential-merge-discipline/README.md** — how stacked PRs through an auto-merge gate silently lose content.
- **public-repo-scrub/README.md** — how stale tags after a history rewrite re-expose scrubbed commits.
- **env-var-verification/README.md** — how a planning-doc variable name caused ~30 minutes of production identity failure.
- **chatterbox/README.md** — why terse mode needs a gauge before a throttle: measure burn at close, activate by trigger, verify the savings.

The generalizing principle: AI-assisted projects fail not because the models are wrong, but because the scaffolding around the models is missing. These skills are scaffolding.

---

## License

These skills are MIT-licensed (see LICENSE in this directory). Fork, adapt, use. Attribution appreciated but not required.
