# session-close — The Ritual That Makes an AI Workforce Resumable

**Origin:** extracted from 100+ AI working sessions building a production consumer AI product — a solo founder working with an AI workforce.

---

## The problem

An AI working session ends and everything in working memory disappears. The code is committed (if you remembered to commit), but the context that makes the code legible is gone: why this approach, what was tried and failed, what remains to be done, what the next session must not do before it reads the state file.

Multiply this across 5 different models (orchestrator, code-gen, adversarial reviewer, visual QA, spec-writer) with no shared memory, across 100+ sessions spanning months. Without a close ritual, every session starts with archaeology. With it, every session starts with orientation.

## The failure that produced this

Three kinds of failures shaped this ritual:

**The archaeology problem.** Early sessions had no status snapshot discipline. The next session would open the changelog, try to reconstruct what "in progress" meant, check the code to see what was actually there versus what the changelog claimed, and spend 15-20 minutes before doing any productive work. At 100 sessions, that's 25-30 hours of wasted session budget.

**The audit trail drift.** An hourly auto-push job ran in the background. In several sessions, the auto-push scooped changes made during the session under its own commit subject ("auto: hourly push") before the session's manual close commit. The next session saw commits in the wrong order, couldn't tell which work was attributed to which session, and once re-applied a change that had already been committed — creating a duplicate that showed up as a merge conflict.

**The near-leak.** One session was working in a context that included references to an employer name (in a compliance scanner's allowlist documentation). The staged diff scan was about to be skipped as "obviously fine." It wasn't — the employer name appeared in a newly added comment that was not a self-referential governance mention. Caught by the scan, removed before commit.

## What the skill does

1. Writes a specific, honest changelog entry — "what changed" names files and systems, not outcomes.
2. Updates a bounded status snapshot (under 5K tokens) the next session reads at boot.
3. Scans the staged diff for forbidden strings before every commit, with explicit handling for self-referential false positives.
4. Checks `git log -2` before adding changes to detect automated commits that may have already captured work.
5. Enforces specific `git add` (no `git add -A` without a diff review).
6. Generates a compact session-starter handoff when the context window is near its limit.

## Key insight

The close ritual's value is not in any single step — it's in the compounding effect across sessions. A team of models that each closes properly can hand off to any other model, cold, and resume in minutes. A team that doesn't close properly spends the first 20% of every session on archaeology. Over 100 sessions, that difference is the difference between a product shipping and not.
