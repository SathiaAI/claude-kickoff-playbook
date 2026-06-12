---
name: env-var-verification
description: Pre-flight procedure for writing and reviewing deployment runbooks that reference environment variables. Enforces git-grep-at-deployed-ref verification so runbook variable names match the code that will actually run. Covers finding the deployed SHA, grep patterns for variable names, and a pre-flight checklist. Use when the user says "write a deployment runbook", "document the env vars for this service", "runbook for deploying this service", "what environment variables does this need", "set up env vars for production", "document the configuration", "the env var in my runbook is wrong", "variable name mismatch in production", or "verify env var names before deploy".
---

# Env Var Verification

Runbooks reference environment variable names. Variable names change in code during development. When the runbook was written against a planning document or an earlier version, and the code at the deployed SHA has moved on, following the runbook configures the wrong variables. The service reads nothing, silently falls back, or fails in a way that takes time to diagnose.

The fix is one step: grep the variable name from the code at the deployed SHA, not from any document. Documents are not ground truth. Code at the deployed commit is.

## When to use

- Writing any runbook, setup guide, or operational document that references environment variable names.
- Reviewing an existing runbook before using it on a new deployment.
- Debugging a service that is not reading configuration it should be receiving.
- Onboarding a new service that was renamed or refactored since the last deploy.

## When NOT to use

- The variable name was copied directly from a `grep` of the current deployed code within the last session — it is already verified.
- The service has not changed since the last verified runbook run.

---

## Step-by-step procedure

### Step 1 — Find the deployed SHA

Before grepping anything, establish exactly which commit is running in production.

**Platform dashboard method (most reliable):**

Most deployment platforms expose the deployed commit hash in their UI or API.

```bash
# Railway (GraphQL API)
curl -s -X POST https://backboard.railway.app/graphql/v2 \
  -H "Authorization: Bearer $RAILWAY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { deployments(input: { projectId: \"YOUR-PROJECT-ID\", serviceId: \"YOUR-SERVICE-ID\" }) { edges { node { id status meta } } } }"
  }' | jq '.data.deployments.edges[0].node.meta.commitHash // "not found"'

# Heroku
heroku releases --app YOUR-APP-NAME --json | jq '.[0].description'

# Vercel (CLI)
vercel ls YOUR-PROJECT --prod --token $VERCEL_TOKEN | head -5

# General: if the platform exposes a /version or /healthz endpoint that echoes the SHA
curl -s https://YOUR-SERVICE-HOST/healthz | jq '.commit // .sha // .version'
```

**Git method (if you know the deployed branch):**

```bash
git fetch origin
git log origin/main -1 --format="%H %s"
```

Record the SHA. All subsequent greps run against this specific commit.

### Step 2 — Grep variable names from the deployed SHA

Never grep from your working copy unless it matches the deployed SHA exactly. Use `git show` or `git grep` scoped to the commit.

```bash
# Check if a variable name exists in the code at the deployed SHA
git grep "ENV_VAR_NAME_HERE" DEPLOYED-SHA -- "*.ts" "*.js" "*.py" "*.go" "*.env.example"

# If you have the deployed SHA checked out locally:
git grep "ENV_VAR_NAME_HERE"

# Check how the variable is read (process.env.X, os.environ.get('X'), etc.)
git show DEPLOYED-SHA:src/config.ts | grep -E "(process\.env\.|os\.environ|getenv)" | head -20

# Find ALL env var reads in a service at the deployed SHA
git grep -E "process\.env\.[A-Z_]+" DEPLOYED-SHA -- "*.ts" "*.js" | grep -oE "process\.env\.[A-Z_]+" | sort -u

# Python equivalent
git grep -E "os\.environ(\[|\.get\()['\"]" DEPLOYED-SHA -- "*.py" | grep -oE "[A-Z_]{3,}" | sort -u
```

**What you are verifying:**

1. The variable name exists in the code at the deployed SHA (not renamed, not removed).
2. The casing is exact — most runtimes treat `MY_VAR` and `my_var` as different variables.
3. The service actually reads the variable at boot (not lazily at first use in a path that may not run).

### Step 3 — Check where the variable is consumed

Reading a variable at boot (during initialization) is safer than reading it lazily. Confirm which applies:

```bash
# Find where the variable is consumed relative to service startup
git show DEPLOYED-SHA:src/index.ts | head -50
# Look for: is the variable read during module load or inside a function that may not run?

# If the service has a config module, check it
git show DEPLOYED-SHA:src/config.ts
```

A variable that is only read inside an infrequently-hit code path may appear to be working when it is actually being ignored. Boot-time reads fail fast; lazy reads fail silently.

### Step 4 — Write the runbook entry with verified names

Only now write the runbook line. Format that makes future verification easy:

```markdown
## Environment Variables

Verified against commit: `DEPLOYED-SHA` on `YYYY-MM-DD`.

| Variable | Required | Purpose | Verified in code |
|----------|----------|---------|-----------------|
| `EXACT_VAR_NAME` | Yes | [What it configures] | `src/config.ts:12` |
| `EXACT_VAR_NAME_2` | No | [What it configures, default if absent] | `src/config.ts:18` |

To re-verify before next deploy:
git grep "EXACT_VAR_NAME" NEXT-DEPLOYED-SHA
```

Including the SHA and file:line reference makes the runbook self-auditing — the next reader can re-run the grep in thirty seconds.

### Step 5 — Pre-flight before using an existing runbook

Before following a runbook that may be stale:

```bash
# 1. Get the current deployed SHA (step 1)
# 2. For each variable name in the runbook, verify it still exists at that SHA
for VAR in VAR_ONE VAR_TWO VAR_THREE; do
  result=$(git grep "$VAR" DEPLOYED-SHA 2>/dev/null | head -1)
  if [ -n "$result" ]; then
    echo "OK: $VAR — $result"
  else
    echo "MISSING: $VAR — not found at DEPLOYED-SHA; runbook may be stale"
  fi
done
```

Any `MISSING` result means the runbook has drifted. Update the runbook before proceeding.

---

## Failure modes

| Failure | What happens | Fix |
|---------|-------------|-----|
| Grepping from a planning document | Variable was renamed in code; service reads nothing; silent misconfiguration | Always grep the deployed SHA; documents are not ground truth |
| Grepping from working copy with uncommitted renames | Runbook written for code that is not deployed yet | Confirm deployed SHA first; grep that exact commit |
| Wrong casing | Runtime treats `My_Var` and `MY_VAR` as different; variable not read | Grep for the exact casing present in the code |
| Variable is read lazily | Service starts healthy; fails when the code path is hit in production | Check boot-time vs. lazy reads in step 3 |
| Runbook not updated after refactor | Next person following the runbook configures the old name | Include deployed SHA in runbook; pre-flight re-verify before each use |

---

## Checklist

- [ ] Deployed SHA identified via platform API or dashboard (not assumed from branch name)
- [ ] Every variable name grepped from that exact SHA (not from docs, not from working copy)
- [ ] Casing verified: grep results show exact match, not case-insensitive
- [ ] Variable confirmed read at boot, not lazily
- [ ] Runbook entry includes the verified SHA and file:line reference
- [ ] If using an existing runbook: pre-flight loop run against current deployed SHA before proceeding

---

## What this skill does NOT do

- Does not manage secrets or key values — it only verifies names. Secret values belong in a secrets manager, not a runbook.
- Does not cover all runtime configuration systems — adjust grep patterns for your framework's config loader (dotenv, Viper, Spring Boot, etc.).
- Does not verify that the value set for a variable is correct, only that the name is what the code expects.
