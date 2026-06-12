# deps-pr-triage — Dependency PR Queue Triage

**Origin:** extracted from 100+ AI working sessions building a production consumer AI product — a solo founder working with an AI workforce.

---

## The problem

Automated dependency bots (Dependabot, Renovate) do excellent work generating PRs. They do no work deciding which ones to merge, in what order, or how to handle the noise an AI review gate introduces. Left unmanaged, the queue grows until you either merge blindly or ignore it until a CVE forces your hand.

The harder problem is that AI-assisted projects add a new failure mode: AI code reviewers confidently flag version-existence claims that are simply wrong, because their training data predates the release.

## The failure that produced this

On one project, a dependency bump PR was held up by an AI review gate that flagged the bumped version as nonexistent. The reviewer's confidence was high. The claim was false — the version had been published; the reviewer's training data was stale. Verifying directly against the npm registry took thirty seconds and confirmed the version was real. Dismissing the comment and merging proceeded without incident.

A second pattern emerged from npm's audit tooling: `npm audit --omit=dev` in npm 10 continued surfacing devDependency vulnerabilities even with that flag set. The audit was being used as a production security gate, so these false alarms created noise and made it harder to spot real issues. The fix was a small script that reads the lockfile's dev flag for each package and filters the audit output accordingly, producing a clean production-only view.

Both failures share a root cause: trusting a tool's output without verifying against the ground truth source — the package registry for version claims, the lockfile for dependency classification.

## What the skill does

1. Sorts the PR queue by semver track (patch / minor / major) before touching anything.
2. Provides registry-verification commands for every ecosystem to disprove AI reviewer version claims in under a minute.
3. Gives a lockfile-based npm audit filter that produces reliable production-only vulnerability output.
4. Enforces sequential, verified merging for patch and minor waves.
5. Hard-blocks major bumps from batch merges and routes them to dedicated upgrade sessions.

## Key insight

AI reviewers are wrong about package version existence at a rate that should be treated as the base case, not the exception. The registry API is the ground truth. Verify before acting, every time.
