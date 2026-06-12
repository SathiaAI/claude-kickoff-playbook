# sequential-merge-discipline — Safe Multi-PR Landing

**Origin:** extracted from 100+ AI working sessions building a production consumer AI product — a solo founder working with an AI workforce.

---

## The problem

Automated merge gates — merge queues, CI-triggered auto-merge, squash-on-green — are designed to help. They become a hazard when multiple PRs are submitted simultaneously, because the gate advances the target branch after each merge without re-evaluating subsequent PRs against the new tip.

The result is a class of failure that is particularly hard to debug: the PR shows as "Merged" in the UI, CI was green, and yet the content is not on the target branch. It merged into a branch that no longer points where it should.

## The failures that produced this

On one project, two PRs were queued through an automated gate in the same session. The first PR merged successfully. The second PR's source branch had been created before the first merge advanced the target branch. When the gate processed the second PR, it merged into a base that had moved on. The content from the second PR was silently lost. It was discovered only when the next session's work was tested and found missing an expected change. The recovery was re-landing the content as a fresh PR — an extra cycle that would have been avoided by merging sequentially.

A related failure occurred separately: an AI agent inferred a PR's source branch from the local checkout state rather than reading it from the GitHub API. The inferred branch was wrong. A push went to the wrong branch and required a force-push to correct. The fix is simple: read `head.ref` from the API or ask; never infer.

## What the skill does

1. Enforces source-branch verification via the API before any push, eliminating wrong-branch errors.
2. Requires strictly sequential PR submission through automated gates — no simultaneous submissions.
3. Provides grep-based tip verification after every merge so "Merged" in the UI is confirmed against actual branch content.
4. Covers recovery when content is found missing: re-land as a fresh PR, do not re-open the orphaned one.
5. Provides a final multi-PR verification sweep to close the merge session cleanly.

## Key insight

The PR UI's "Merged" badge is not a content guarantee. The only reliable confirmation that a change landed is finding the expected content at the target branch's current tip. Hash-verify after every merge — it takes thirty seconds and catches silent losses before they become archaeology problems.
