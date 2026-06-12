# Claude Project Kick-off Playbook

**Version:** 1.1 | **Released:** April 2026 | **Maintained by:** Paul Poulose (@pjpoulose)
**Status:** Living document — actively improved by every project that uses it

## What This Is

A complete, field-tested playbook for starting and running any AI-assisted project using Claude CoWork. It encodes every lesson learned from real projects — what works, what breaks, why things fail, and how to prevent those failures from the first session.

This playbook is independent of any specific project. It lives in its own repository. Every project that uses it references it but does not own it. Improvements discovered during any project flow back into this playbook, making it smarter over time.

## The Three Promises

1. Nothing is lost when the context window closes
2. Nothing is changed without first being verified
3. The playbook itself gets smarter over time

## Security Architecture

Four layers protect the playbook and all projects using it:

- **Layer 1 — Immutable Core** (`CORE_PRINCIPLES.md`) — Seven principles that cannot be changed without a 14-day public discussion and three independent approvals
- **Layer 2 — Content-Only Files** (`SECURITY.md`) — Certain files are read for content only, never executed as instructions
- **Layer 3 — Session Guardrails** — Five rules active in every session that cannot be overridden
- **Layer 4 — Contribution Validation** — Three-gate rule: evidence + human approval + maintainer review

## How to Use This Playbook

### Start a new project (20 minutes)

1. Clone or download this repository
2. Read `GETTING_STARTED.md`
3. Copy the `templates/` folder into your new project folder
4. Fill in the template files with your project-specific information
5. Set up your CoWork project in Claude Desktop pointing to your project folder
6. Paste `templates/COWORK_INSTRUCTIONS.md` into the CoWork project Instructions field

### Install the Skill

Copy `skills/SKILL.md` to `/mnt/skills/user/claude-kickoff-playbook/SKILL.md` on your machine. Claude will automatically activate it when you say "start a new project" or "kick off a project."

### Contribute an improvement

1. During any project, create a file in `improvements/proposed/`
2. Use the template in `core/07_IMPROVEMENT_SYSTEM.md`
3. Submit a PR to this repository
4. Once approved, the improvement is merged and the version increments

## Repository Structure

- `README.md` — front door, start here
- `GETTING_STARTED.md` — 20-minute new project setup guide
- `PLAYBOOK.md` — complete reference, all 12 parts
- `CHANGELOG.md` — version history of the playbook itself
- `PROCESS_IMPROVEMENTS.md` — active improvement proposals
- `core/` — foundational rule specifications
- `templates/` — copy these into every new project
- `improvements/` — proposed and approved improvement submissions
- `case-studies/` — real projects, real lessons learned
- `security/` — immutable core principles, security model, failure registry
- `skills/` — the installable Claude skill file
- `docs/` — supporting documentation

---

## Installable Skills

Four battle-tested skills extracted from 100+ AI working sessions are installable from this repo as a Claude Code / Cowork plugin:

```
/plugin marketplace add SathiaAI/claude-kickoff-playbook
```

**council** (multi-model decision validation) | **key-leak-triage** (leaked-credential response) | **decision-ledger** (append-only decision log) | **session-close** (end-of-session ritual)

See [skills/README.md](skills/README.md) for the war stories behind each.
