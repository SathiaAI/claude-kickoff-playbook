---
name: deps-pr-triage
description: Structured triage procedure for automated dependency-bump PRs (Dependabot, Renovate, or similar). Covers batching strategy, AI-reviewer false-positive handling, registry verification, and safe merge sequencing. Use when the user says "triage my dependency PRs", "Dependabot PRs are piling up", "Renovate opened 20 PRs", "should I merge these dep bumps", "AI reviewer flagged this version as nonexistent", "npm audit is giving me noise", "safe to merge minor dependency updates", "how do I handle major version bumps", "batch dependency updates", or "dependency PR queue".
---

# Deps PR Triage

Automated dependency bots generate volume. Merging blindly causes breakage; ignoring indefinitely creates real security debt. The procedure below drains the queue systematically — without merging anything you haven't verified and without being paralyzed by AI reviewer false positives.

## When to use

- A batch of dependency-bump PRs has accumulated and needs routing decisions.
- An AI code reviewer has flagged a dependency bump with a version-existence claim you want to verify.
- You are deciding whether a major version bump is safe to merge now or should be held.
- You want a reliable production-only audit (ignoring dev-dependency noise).

## When NOT to use

- Individual hand-crafted dependency changes — those go through normal code review.
- Security patches that need immediate, individual attention — triage those first, outside this batch flow.

---

## Step-by-step procedure

### Step 1 — Sort the queue

Classify every open dependency PR into one of three tracks before touching any of them:

| Track | Criteria | Merge posture |
|-------|----------|--------------|
| **Patch** | Semver patch bump (x.y.Z) | Merge after CI green; no manual review needed |
| **Minor** | Semver minor bump (x.Y.0) | Merge after CI green; skim changelog for breaking notes |
| **Major** | Semver major bump (X.0.0) | Hold for a dedicated upgrade session; never merge in a batch |

Group all patches into one merge wave, all minors into a second wave, and never mix majors into either.

### Step 2 — Handle AI-reviewer version claims

If an AI code reviewer comments that "this version doesn't exist" or "I cannot find version X.Y.Z in the registry":

**Treat this as a near-certain false positive.** AI reviewers have a training data cutoff. Packages release constantly. A reviewer trained months ago will flag versions that exist today.

Verify the tag directly against the registry before acting:

```bash
# npm / Node.js ecosystem
npm view PACKAGE-NAME@VERSION version
# Returns the version string if it exists; error if it does not

# Alternative: check npmjs.com directly
curl -s https://registry.npmjs.org/PACKAGE-NAME/VERSION | jq '.version // "NOT FOUND"'

# Python / PyPI
curl -s https://pypi.org/pypi/PACKAGE-NAME/VERSION/json | jq '.info.version // "NOT FOUND"'

# Ruby / RubyGems
curl -s https://rubygems.org/api/v2/rubygems/PACKAGE-NAME/versions/VERSION.json | jq '.version // "NOT FOUND"'
```

**If the registry confirms the version exists:** dismiss the reviewer's comment as a false positive. Merge if CI is green. Do not open a follow-up issue for a false positive — it creates noise.

**If the registry cannot find the version:** the bump PR itself is malformed. Do not merge. Flag the dependency bot configuration or the PR source for investigation.

### Step 3 — Run a reliable production-only audit

`npm audit --omit=dev` in npm 10 is unreliable — devDependency vulnerabilities can still surface even with that flag, producing noise in CI.

Use a lockfile-based filter instead:

```bash
# Get the full audit JSON
npm audit --json > /tmp/audit-full.json

# Filter to production-only packages using the lockfile dev flag
node - <<'EOF'
const audit = JSON.parse(require('fs').readFileSync('/tmp/audit-full.json', 'utf8'));
const lock  = JSON.parse(require('fs').readFileSync('package-lock.json', 'utf8'));

const devPkgs = new Set(
  Object.entries(lock.packages || {})
    .filter(([, meta]) => meta.dev)
    .map(([name]) => name.replace(/^node_modules\//, ''))
);

const prodVulns = Object.entries(audit.vulnerabilities || {})
  .filter(([name]) => !devPkgs.has(name));

if (prodVulns.length === 0) {
  console.log('No production vulnerabilities found.');
} else {
  prodVulns.forEach(([name, v]) => console.log(`${name}: ${v.severity} — ${v.via?.[0]?.title ?? 'see npm audit'}`));
}
EOF
```

Only act on vulnerabilities the script surfaces. DevDependency vulnerabilities are a code quality concern, not a production security concern.

### Step 4 — Merge patch and minor waves sequentially

Merge one wave at a time, waiting for CI to complete between waves:

1. Merge all patch PRs. Wait for the target branch CI to pass after each merge before merging the next. Do not stack patch merges through an automated gate simultaneously — see the failure mode below.
2. After all patch merges are green: merge minor PRs one at a time, checking changelogs first.
3. After each wave: verify the target branch tip contains the expected content (grep for a changed version string, don't just trust the PR UI).

### Step 5 — Hold majors

Major bumps break builds at an unpredictable frequency and should never be merged in a batch wave:

- TypeScript major versions change compiler behavior and often require code changes.
- Framework majors (React, Vue, Angular, Next.js, etc.) routinely require migration work.
- Node.js type-definition packages (`@types/*`) track engine majors and break APIs.

Schedule a dedicated upgrade session per major dependency. Treat it as a feature, not a maintenance task.

---

## Failure modes

| Failure | What happens | Fix |
|---------|-------------|-----|
| Acting on AI reviewer version claim | Version exists; you reject a valid bump or file a bogus issue | Always verify via registry API before acting |
| Using `npm audit --omit=dev` for prod security decisions | DevDep vulns still appear; CI fails on non-issues | Use the lockfile-based filter in step 3 |
| Batch-merging through an automated gate | Stacked PRs can lose content if the gate advances without them (see sequential-merge-discipline skill) | Merge sequentially; verify tip between merges |
| Merging a major in a patch wave | Build breaks on a Friday | Sort by semver track in step 1; never merge majors in waves |
| Skipping changelog review on a minor | Minor contained a breaking note in the release | Skim the changelog for "BREAKING", "deprecated", or "removed" before merging minor bumps |

---

## Checklist

- [ ] Queue sorted by semver track (patch / minor / major)
- [ ] Any AI reviewer version claims verified against registry API
- [ ] Production audit run with lockfile-based filter (not raw `--omit=dev`)
- [ ] Patch wave merged sequentially, CI verified between each merge
- [ ] Minor wave merged with changelog review, sequentially
- [ ] Major bumps held for dedicated upgrade sessions
- [ ] Target branch tip verified after each wave (grep, don't trust PR UI)

---

## What this skill does NOT do

- Does not automatically merge PRs — merge decisions remain human-gated.
- Does not cover major version migration work — that is a feature-level task.
- Does not replace reading security advisories for high-severity vulnerabilities — high severity patches should be prioritized individually, not batched.
