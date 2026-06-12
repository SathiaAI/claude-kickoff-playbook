---
name: key-leak-triage
description: Structured triage and rotation procedure for a leaked credential or API key. Enforces triage-before-revoke discipline to avoid turning a security incident into a production outage. Use when the user says "I leaked a key", "found a key in a commit", "API key is in git history", "credential exposed", "secret in a log", "key was in a screenshot", "rotate a compromised key", "key appeared in a PR diff", "secret scanner flagged a key", or "I accidentally pushed a secret".
---

# Key Leak Triage

**Core rule: triage before revoke.** Panic-revoking a leaked key before you know who uses it can cause an outage worse than the leak itself. The procedure below takes 5-20 minutes. The outage it prevents can take hours.

## When to skip triage and revoke immediately

- The key is **actively being abused** right now (you see unfamiliar API calls in the provider's usage logs in real time).
- The provider has already suspended the key.
- The exposure window is hours or days and the blast radius is too large to enumerate.

In these cases: revoke first, accept the outage, then work through the recovery. Come back to step 4 (abuse check) and step 6 (prevent) when the fire is out.

## When NOT to use this skill

- Key was never committed, logged, or shared — it is still safe.
- Key is intentionally public (e.g., a publishable/anonymous API key by design).

---

## Step-by-step procedure

### 1. FREEZE — do not revoke yet

Stop. Before touching the key:

- Note the **exact location** where the leak occurred (commit SHA, file path, log file, screenshot, PR URL, Slack message).
- Note the **exposure window**: when was it first visible, to whom, on what surface (private repo, public repo, shared Slack channel, public log endpoint)?
- If it is in git history, note whether the repo is public or private.

### 2. FIND CONSUMERS — who actually uses this key?

This is the step most people skip. UI labels on keys are NOT reliable indicators of what runtime systems consume them.

Run all of the following:

```bash
# Search by the key value itself (first 8 chars is enough if the full key risks re-logging)
grep -r "sk-FIRST-8-CHARS" . --include="*.env" --include="*.json" --include="*.yaml" \
  --include="*.yml" --include="*.ts" --include="*.js" --include="*.py" \
  --include="*.sh" --include="*.toml" -l

# Search by the key's name/label as it appears in config
grep -r "KEY_NAME_HERE" . -l

# Check CI configs
grep -r "KEY_NAME_HERE" .github/ .gitlab-ci.yml .circleci/ -l 2>/dev/null

# Check deployment env var manifests (adjust paths for your infra)
grep -r "KEY_NAME_HERE" infra/ deploy/ k8s/ platform/ fly/ -l 2>/dev/null
```

Also check:
- Your deployment platform's environment variable list for every service (not just the one you suspect).
- Any secrets manager or vault entries referencing this key name.
- Cron jobs, batch runners, and scheduled tasks — they often use keys not in your main service configs.

### 3. CLASSIFY — local-only or runtime-critical?

**Classification A — local-only:**
The key appears only in local scripts, developer tooling, or one developer's machine. No production service consumes it.

Action: rotate at leisure. Revoking immediately is safe because nothing in production will break.

```bash
# Provision the new key at the provider first
# Update your local scripts / .env.local
# Then revoke the old key
# Verify old key is 401
```

**Classification B — runtime-critical:**
The key is consumed by a running production service, a CI pipeline that gates deployments, or any automated system that runs without human intervention.

Action: rotate-then-revoke. Never revoke before the replacement is live.

```bash
# Step 1: Provision the replacement key at the provider
NEW_KEY=sk-REPLACE-WITH-NEW-KEY

# Step 2: Deploy the new key to every consumer found in step 2
# (update env vars in your deployment platform, restart services)

# Step 3: Verify every consumer is healthy with the new key
# (check health endpoints, run smoke tests, review deployment logs)

# Step 4: ONLY THEN revoke the old key

# Step 5: Verify old key returns 401
curl -H "Authorization: Bearer sk-REPLACE-WITH-OLD-KEY" \
  https://provider.example.com/v1/models
# Expect: 401 Unauthorized

# Step 6: Verify services still healthy after revoke
```

### 4. CHECK ABUSE — review usage logs

At the provider's dashboard or API, pull usage for the exposed key across the full exposure window:

- Look for calls from unfamiliar IP addresses, regions, or user agents.
- Look for model aliases, endpoint paths, or usage volumes inconsistent with your normal patterns.
- Look for any calls that occurred AFTER you stopped using the key normally (a clear abuse signal).

Document what you find. If abuse is confirmed, treat this as a security incident, not just a rotation.

### 5. REVOKE AND VERIFY

```bash
# Revoke the old key at the provider (UI or API)

# Verify: old key is now rejected
curl -H "Authorization: Bearer sk-REPLACE-WITH-OLD-KEY" \
  https://provider.example.com/v1/models
# Must return 401

# Verify: production services are still healthy
# (check health endpoints, recent successful API calls in logs)
```

Do not skip the health check after revocation. Sometimes the new key deployment had a silent error and services were actually still using the old key.

### 6. PREVENT — close the leak path

#### If the key is in git history:

```bash
# Rewrite history (destructive — coordinate with team first)
git filter-repo --path-glob '*.env' --invert-paths
# or use git filter-repo --replace-text with a mapping file

# Force push
git push origin --force --all

# CRITICAL: retag from the new main
# Stale tags point to the old commits and re-expose scrubbed history
git tag -d v1.0.0   # delete old tag
git push origin :refs/tags/v1.0.0  # delete remote tag
git tag v1.0.0 HEAD  # retag from current clean tip
git push origin --tags
```

After history rewrite, all collaborators must re-clone or reset their local copies.

#### Prevent recurrence:

1. Add the file or directory that leaked to `.gitignore`.
2. Add a secret scanner to CI (e.g., gitleaks with full-history mode on first run, line-only mode on subsequent runs).
3. Add a pre-push hook that runs the scanner locally before any push reaches the remote.
4. If the key was in a runner script output: add the runner's output directory to `.gitignore` and document this in the runner's README.

---

## Failure modes

| Failure | What happens | Fix |
|---------|-------------|-----|
| Revoking before finding all consumers | Production outage | Always run step 2 before step 5 |
| Trusting the UI label | Missed consumer causes outage after revoke | Grep the actual configs, not the label |
| Skipping health check after revoke | Silent deploy failure means services were using old key all along | Always verify services healthy after revoke |
| Forgetting to retag after history rewrite | Stale tag re-exposes scrubbed commit | Retag from new HEAD immediately after rewrite |
| New key has different scopes than old key | Services fail with 403 (permission denied) not 401 | Provision new key with same scopes; test before revoking old |

---

## What this skill does NOT do

- Does not revoke keys directly — only the key provider's UI or API can do that.
- Does not handle certificate rotation — different procedure for TLS certificates.
- Does not escalate to security team or file incident reports — do that in parallel per your org's policy.
