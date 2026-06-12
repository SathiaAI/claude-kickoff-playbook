# env-var-verification — Runbook Env Var Name Verification

**Origin:** extracted from 100+ AI working sessions building a production consumer AI product — a solo founder working with an AI workforce.

---

## The problem

Deployment runbooks reference environment variable names. Variable names get renamed during development — sometimes in a single refactor, sometimes gradually as a feature evolves. Runbooks drafted against planning documents or design notes accumulate these renames silently, because the documents are not the code.

When the runbook is followed in production, the service is configured with the old name. The service starts, appears healthy, and silently falls back to defaults or reads nothing. Depending on what the variable controls, this can mean degraded functionality, wrong behavior, or a complete identity resolution failure — all without an obvious error.

## The failure that produced this

On one project, a runbook referenced a variable name drawn from a planning document. The variable had been renamed in code before the deploy — a pilot-identity configuration variable that had been updated during feature development. Following the runbook in production configured the old name. The service started cleanly and appeared healthy. The identity resolution it was supposed to enable did not work. Diagnosing and correcting the misconfiguration took approximately thirty minutes.

The fix was straightforward: `git grep` the variable name from the deployed commit's code. The code had the new name. The runbook had the old name. The planning document, which the runbook was drafted from, had never been updated to reflect the rename.

The lesson: planning documents are not ground truth. The code at the deployed commit is.

## What the skill does

1. Provides platform-specific commands for finding the deployed SHA before grepping anything.
2. Provides grep patterns scoped to the deployed commit — not the working copy, not a document.
3. Checks whether variables are read at boot vs. lazily, to distinguish between fast-fail and silent-fail configurations.
4. Produces runbook entries that include the verified SHA and file:line reference, making future re-verification a one-command operation.
5. Provides a pre-flight loop for re-verifying any existing runbook against the current deployed SHA before following it.

## Key insight

The thirty minutes lost to a variable-name mismatch is entirely avoidable. One `git grep` against the deployed SHA takes under a minute. The discipline is: never write a variable name into a runbook from a document; only write it after grep confirms it in the code.
