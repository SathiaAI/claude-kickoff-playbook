---
name: public-repo-scrub
description: Pre-publish scrub procedure for moving content from a private workspace to a public repository without leaking internal references, real pricing, employer names, or unverifiable claims. Covers file-level grep, claim softening, byte-verification after push, and tag discipline after history rewrites. Use when the user says "publish this to a public repo", "open-source this", "make this public", "push to the public repo", "scrub before publishing", "check for leaks before open-sourcing", "clean up before going public", "remove internal references", "rewrite git history to remove a secret", or "tags after history rewrite".
---

# Public Repo Scrub

Publishing from a private workspace to a public repository is a one-way gate. What goes public stays in caches, forks, and third-party mirrors long after deletion. The scrub procedure below is the last line of defense before crossing that gate.

## When to use

- Any time content produced in a private workspace is being moved to a public repository.
- Before open-sourcing internal tooling, skills, scripts, or documentation.
- After a git history rewrite that removed a leaked secret — the tag step is easy to miss and re-exposes the scrubbed commits.

## When NOT to use

- Publishing between two private repositories with equivalent access controls.
- Content that was authored specifically for public consumption from the start (no private context to scrub).

---

## Step-by-step procedure

### Step 1 — Grep ALL files, not just the ones you authored

The files you wrote are not the only files at risk. Dependencies, templates, example blocks, and automatically generated files all carry content. Grep the entire directory tree.

```bash
# Run from the root of the directory being published
# Customize PATTERN-LIST for your project's internals

grep -rniE \
  "(YOUR-EMPLOYER|YOUR-COMPANY|INTERNAL-HOSTNAME|INTERNAL-PROJECT-NAME|api\.internal\.|\.internal\b)" \
  . \
  --exclude-dir=.git \
  --exclude-dir=node_modules \
  --include="*.md" --include="*.ts" --include="*.js" \
  --include="*.json" --include="*.yaml" --include="*.yml" \
  --include="*.sh" --include="*.py" --include="*.toml" \
  --include="*.txt" --include="*.env.example"

# Separately grep for pricing (real dollar amounts in example blocks are a common miss)
grep -rniE '\$[0-9]+(/mo|/month|/yr|/year|per seat|/user)' . --exclude-dir=.git --exclude-dir=node_modules

# Separately grep for email addresses and personal identifiers
grep -rniE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' . --exclude-dir=.git --exclude-dir=node_modules

# Separately grep for API key patterns
grep -rniE '(sk-[a-zA-Z0-9]{20,}|Bearer [a-zA-Z0-9._-]{20,})' . --exclude-dir=.git --exclude-dir=node_modules
```

**Every match must be resolved before proceeding.** Resolution options:
- Remove the reference entirely.
- Replace with a generic placeholder (`YOUR-ORG`, `YOUR-HOSTNAME`, `example.com`).
- Replace with a clearly fictional value that cannot be mistaken for real data.

Do not proceed to step 2 with any unresolved match.

### Step 2 — Audit war stories and claims for verifiability

Content drafted in a private context often contains claims that felt precise in context but are not verifiable (or are overclaiming) when extracted:

- **Specific numbers without a public source** — if you can't cite a public reference, soften the claim. "Saved hours" is better than "saved 4.7 hours per session" if the figure isn't publicly documented.
- **"We always do X"** — generalize to "on one project..." or "in practice..." unless X is definitively always true.
- **Internal issue IDs, ticket numbers, PR numbers** — these are meaningless externally and may hint at internal tooling. Remove or generalize.
- **Session or build labels** ("Session 42", "Build 7c2f", internal run identifiers) — remove; they reveal internal state.
- **References to private infrastructure** — hostnames, internal URLs, Railway/Supabase/platform project IDs. Replace with `YOUR-GATEWAY-HOST`, `YOUR-PROJECT-ID`, etc.

Read every war story as if you are an external reader with no internal context. If a claim requires internal context to understand or verify, rewrite it.

### Step 3 — Push to the public repository

```bash
# Verify the remote is what you expect (public, correct repo)
git remote -v

# Push
git push origin main

# Immediately fetch the push back and byte-verify
git fetch origin main
git diff HEAD origin/main
# Must produce no output — local and remote are identical
```

If the diff is non-empty, the push did not land cleanly. Investigate before announcing or linking the repo.

### Step 4 — Byte-verify the pushed content

After push, spot-check key files by fetching them via the raw URL:

```bash
# GitHub raw URL pattern
curl -s "https://raw.githubusercontent.com/OWNER/REPO/main/PATH/TO/FILE" \
  | sha256sum

# Compare to local
sha256sum PATH/TO/FILE
# Hashes must match
```

Do this for any file that was edited in the scrub (modified content is most at risk of a partial push). If hashes differ, the remote has different content than your local. Investigate before the repo is indexed or shared.

### Step 5 — After any history rewrite, audit and retag

If you rewrote git history to remove a leaked secret (using `git filter-repo` or similar), stale tags pointing to old commits re-expose the scrubbed history. This step is easy to miss.

```bash
# List all tags and where they point
git tag -l | xargs -I{} git log -1 --format="%D %H" {}

# For each tag that points to a pre-rewrite commit:
# Delete the local and remote tag, then recreate it at the new tip

git tag -d TAG-NAME
git push origin :refs/tags/TAG-NAME   # delete remote tag

git tag TAG-NAME HEAD                  # recreate at new clean tip
git push origin --tags                 # push the new tag

# Verify: the tag now points to a post-rewrite commit
git log -1 --format="%H" TAG-NAME
# This hash must NOT appear in the pre-rewrite history you scrubbed
```

Repeat for every tag in the repository. After a history rewrite, all collaborators must re-clone or hard-reset their local copies — stale local clones retain the pre-rewrite commits.

---

## Failure modes

| Failure | What happens | Fix |
|---------|-------------|-----|
| Grepping only authored files | Internal reference in a template or generated file goes public | Grep the entire tree; no exclusions beyond .git and node_modules |
| Missing pricing in example blocks | Real pricing appears in a code or config example | Dedicated dollar-amount grep as a separate pass |
| Overclaimed war stories | Specific numbers or session references confuse or mislead external readers | Soften to verifiable generics; read as an external reader |
| Skipping byte-verification after push | Content mismatch between local and remote goes unnoticed | sha256sum comparison on scrubbed files after push |
| Forgetting to retag after history rewrite | Stale tag re-exposes scrubbed commit; scrapers and mirrors index it | Audit all tags immediately after rewrite; delete and recreate from new HEAD |
| Not notifying collaborators after history rewrite | Collaborators push from stale local clone; scrubbed commits re-enter history | Announce the rewrite; require re-clone or hard reset before any push |

---

## Checklist

- [ ] Full-tree grep for internal names, hostnames, and project references — all matches resolved
- [ ] Dedicated grep for real pricing (dollar amounts in examples)
- [ ] Dedicated grep for email addresses and personal identifiers
- [ ] Dedicated grep for API key patterns
- [ ] All war stories and claims read as an external reader — overclaims softened
- [ ] All internal issue IDs, session labels, and build identifiers removed
- [ ] Pushed to public remote; `git diff HEAD origin/main` returns empty
- [ ] sha256sum byte-verification on scrubbed files
- [ ] If history was rewritten: all tags audited, stale tags deleted and recreated at new HEAD
- [ ] If collaborators exist: notified to re-clone after any history rewrite

---

## What this skill does NOT do

- Does not remove content from third-party caches or forks that already indexed it — once public, assume it was seen.
- Does not cover GDPR or legal review of published content — consult your legal process for that.
- Does not scan binary files — inspect those manually if they might contain embedded text.
