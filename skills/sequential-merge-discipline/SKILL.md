---
name: sequential-merge-discipline
description: Safe discipline for landing multiple PRs through an automated merge gate. Prevents silent content loss from stacked merges, enforces hash-verification of target branch tips, and covers source-branch identification before pushing. Use when the user says "I need to merge several PRs", "landing multiple PRs", "stacked PRs through the merge gate", "merge wave", "queue of PRs to land", "verify the branch after merging", "what branch is this PR targeting", "did my merge actually land the content", "content missing after merge", or "PR went through the gate but the code isn't there".
---

# Sequential Merge Discipline

Automated merge gates — CI-triggered auto-merge, merge queues, branch protection with auto-squash — are powerful. They are also silent about content loss when they receive stacked PRs. The procedure below prevents the most common and costly merge-wave failure: content that appears merged but is actually gone.

## When to use

- You are landing two or more PRs against the same target branch in the same session.
- Any PR has an auto-merge gate (merge queue, CI-required merge, or squash-on-green automation).
- You are collaborating with an AI agent that may infer branch names from local state rather than the remote.

## When NOT to use

- A single PR with no related follow-on PRs — standard merge; no special procedure needed.
- PRs against different target branches that have no dependency on each other.

---

## Step-by-step procedure

### Step 1 — Verify each PR's source branch before any push

Do not infer a PR's source branch from your local checkout state or any local variable. Read it from the API or ask.

```bash
# GitHub CLI — shows head.ref (source branch) directly
gh pr view PR-NUMBER --json headRefName,baseRefName,title

# GitHub REST API
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/OWNER/REPO/pulls/PR-NUMBER" \
  | jq '{head: .head.ref, base: .base.ref, title: .title}'
```

If the source branch does not match what you expect, stop. Do not push to the wrong branch — a mismatched push on one occasion required a force-push to correct, introducing history noise. Confirm before acting.

### Step 2 — Merge one PR at a time

Never send two or more PRs through an automated gate simultaneously.

The failure mode: PR-A and PR-B both target `main`. PR-A merges. If PR-B's source branch was created off the pre-merge `main`, the gate may advance `main` via PR-A while PR-B is still queued. PR-B then merges into a branch that no longer points where it should, silently losing the delta between PR-A's merge commit and the base PR-B was diffed against.

Sequence:

1. Merge PR-A. Wait for the gate to report the merge as complete and for CI on the target branch to go green.
2. Verify the target branch tip (step 3).
3. Only then trigger PR-B.

If you have N PRs to land, this takes N sequential cycles. The overhead is real; the alternative (silent content loss requiring a re-land) is worse.

### Step 3 — Hash-verify the target branch tip after every merge

Do not trust the PR UI's "Merged" badge as confirmation that the content landed. Verify the target branch tip directly.

```bash
# Get the current tip of the target branch
git fetch origin main
git log origin/main -1 --format="%H %s"

# Confirm the expected content is present
git grep "SOME-UNIQUE-STRING-FROM-THE-PR" origin/main
# If this returns a match, the content is present.
# If it returns nothing, the content did not land — see failure modes below.

# Alternative: check a changed file's content at the tip
git show origin/main:path/to/changed-file | head -20
```

Choose a grep target that is unique to the PR's changes — a new function name, a new config key, a specific string added in the diff. Avoid generic strings that might match unrelated content.

### Step 4 — Handle a missing-content finding

If step 3 reveals the expected content is absent after a "successful" merge:

1. Check what commit the target branch tip actually contains: `git log origin/main -5 --oneline`
2. Check whether the PR's source branch still exists: `git ls-remote origin PR-SOURCE-BRANCH`
3. If the source branch is gone and main doesn't have the content, the merge silently dropped it. Re-land as a fresh PR off the current tip of main.
4. Do not re-open the old PR — it merged into an orphaned state. Open a new PR with the same content rebased onto the current tip.

### Step 5 — After all PRs are landed

Do a final tip verification covering all landed PRs:

```bash
# Confirm all expected content strings are present at the final tip
git fetch origin main

for PATTERN in "string-from-pr-1" "string-from-pr-2" "string-from-pr-3"; do
  result=$(git grep "$PATTERN" origin/main 2>/dev/null | head -1)
  if [ -n "$result" ]; then
    echo "OK: $PATTERN"
  else
    echo "MISSING: $PATTERN — investigate before closing"
  fi
done
```

Only close the merge session once all patterns confirm present.

---

## Failure modes

| Failure | What happens | Fix |
|---------|-------------|-----|
| Stacked PRs through auto-merge gate | Second PR merges into orphaned base; content silently lost | Sequential only — never trigger PR-B until PR-A is confirmed at tip |
| Trusting PR UI "Merged" badge | UI says merged; tip doesn't have the content | Hash-verify tip with grep after every merge |
| Inferring source branch from local checkout | Push goes to wrong branch; requires force-push correction | Read head.ref from API or ask; never infer from local state |
| Re-opening a stale PR after lost merge | PR base is now behind; merge conflict or wrong diff | Re-land as a fresh PR rebased onto current tip |
| Generic grep target in verification | False positive match hides that the real change is missing | Use a unique string specific to the PR's diff |

---

## Checklist

- [ ] Every PR's source branch confirmed via API (not inferred from local state)
- [ ] Merge order documented before any gate is triggered
- [ ] PR-A merged and CI green on target branch before triggering PR-B
- [ ] Target branch tip hash-verified after each merge (grep unique string)
- [ ] No stacked simultaneous submissions to the merge gate
- [ ] Final verification covers all landed PRs before closing the session

---

## What this skill does NOT do

- Does not automate the merge — merge decisions remain human-gated.
- Does not recover lost commits from a completed merge — if content was lost, re-land it as a fresh PR.
- Does not cover conflicts between PRs that touch the same files — resolve those by rebasing before merging, outside this procedure.
