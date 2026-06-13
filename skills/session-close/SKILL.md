---
name: session-close
description: End-of-session ritual for AI working sessions — writes changelog with a session-economics footer (tokens, cost, contract score), updates status snapshot, scans staged diff for forbidden strings, checks audit trail integrity, writes a next-session contract with binary acceptance criteria and a scope gate, and optionally generates a session-starter handoff. Use when the user says "close the session", "wrap up", "end of session", "session close", "commit and close", "write the changelog entry", "update the status snapshot", "generate a session starter", "session handoff", "scan before commit", "what did this session cost", "session economics", "plan the next session", or "what do we need to do before we close".
---

# Session Close

The next session boots cold. Whatever isn't written down is lost. Whatever is committed with the wrong attribution misleads the next agent. Whatever gets pushed without a forbidden-string scan can leak something you'll regret. This ritual takes 10-15 minutes and makes the next session start in minutes, not hours.

## When to use

- At the end of any substantial working session (not a 5-minute clarification, but anything that produced changes, decisions, or findings).
- Before ending a session that is approaching context limit — generate the handoff file first.
- Any time a session produced code, configuration, or documentation that needs committing.

## When NOT to use

- The session produced no substantive changes (research only, no decisions, no code).
- A formal close was already run and nothing has happened since.

---

## Step-by-step procedure

### Step 1 — Write the changelog entry

Append a new entry to your project's changelog file. Format:

```markdown
## Session YYYY-MM-DD [optional-track-label]

**Headline:** [One sentence: the single most important thing that happened]

**What changed:**
- [Change 1 — be specific: file/component/system modified, not "made improvements"]
- [Change 2]
- [Change 3]

**Verification state:**
- Build: [pass / fail / not run — reason if not run]
- Tests: [pass / fail / N/A — reason]
- Linting: [pass / fail / N/A]

**NEEDS-OWNER:**
- [Item 1 requiring human decision or action — empty if none]

**Not done / deferred:**
- [Item 1 — why deferred, what depends on it]

**Economics:** tokens ~[N]K EST-LOW (+[M]K MEASURED API) · cost ~$[X] EST / $[Y] MEASURED · calib ratio [R] (checked [date])
**Contract score:** [N]/[M] AC met · goal: met | partial | not met — [one sentence]
**Anomalies:** [none | one line: rework loop on X / pivot to Y (reason) / tool failure burned ~Z tokens — include the exact error message]
```

Rules:
- "What changed" must be specific enough that someone reading it can grep the repo and find the relevant commit. "Updated auth logic" is not specific. "Added RLS policy to vault_entries table to restrict cross-family reads" is specific.
- Verification state must be honest. Do not write "pass" if you did not actually run the check.
- NEEDS-OWNER items are surfaced to the human before closing. Do not silently carry them into the next session.

Economics footer rules (3 lines, under 30 seconds to write):
- **MEASURED vs ESTIMATED are never summed into one number.** MEASURED = usage fields actually returned by metered API calls made during the session (most gateways and model APIs report them). ESTIMATED = everything else.
- For interactive agent sessions with no usage API, estimate output tokens as transcript characters ÷ 4 and label it **EST-LOW** — this heuristic misses system prompts, tool calls, and retries, and typically underestimates by 30-50%. Never fabricate precision; "unmeasured" is an acceptable value.
- **Calibration ratio:** weekly (not per-session), spot-check your EST numbers against any real dashboard you have (gateway spend logs, provider console). Record the ratio (e.g., `calib ratio 1.4x, checked 2026-06-12`) and apply it to future estimates. The goal is trend detection — is cost creeping up? are sessions bloating? — not billing-grade accounting.
- **Contract score is scored against the contract written at the END of the PREVIOUS session** (see step 7), not against whatever the session drifted into. If the session pivoted, the pivot and its reason go in Anomalies — a justified pivot is not a failure, but an unexplained drift is.
- The goal verdict is binary-ish on purpose: met / partial / not met plus one sentence. No 1-5 self-scores — an agent grading its own work on a subjective scale produces theater, not signal.
- Anomalies stay qualitative. One line. If a tool failed, include the exact error message so the human can audit the claim later. Never score or grade anomalies — scored flags invite blame-shifting.

### Step 2 — Update the status snapshot

The status snapshot is a bounded file (keep it under 5K tokens) that the next session reads at boot. It is a snapshot, not a narrative. Update it in place.

Format:

```markdown
# Status Snapshot — Last updated YYYY-MM-DD

## Phase
[Current build phase / sprint / milestone in one line]

## Blockers
- [Blocker 1 — what's blocked, what resolves it]
- (empty if none)

## In-flight work
- [Work stream 1: what state it's in, next action]
- [Work stream 2: what state it's in, next action]

## Key context for next session
- [Fact 1 the next session needs to know that isn't obvious from the code]
- [Fact 2]

## Last verified state
- [Service or component]: [status and timestamp of last verified check]
```

**Archive trigger:** if the snapshot file has grown past 5K tokens (or ~150 lines), move the old content to a dated archive entry and reset the snapshot. Size discipline prevents the snapshot from becoming a second changelog.

### Step 3 — Forbidden-string scan on staged diff

Before committing, run this scan on the staged diff. Adjust the pattern list for your project.

```bash
# Stage your changes first, then scan the diff
git diff --cached > /tmp/staged.diff

# Scan for forbidden patterns (customize this list for your project)
grep -inE \
  "(sk-[a-zA-Z0-9]{20,}|YOUR-EMPLOYER-NAME|INTERNAL-HOSTNAME|INTERNAL-PROJECT-REF)" \
  /tmp/staged.diff

# If matches found: review each match
# Self-referential matches (the blocklist file mentioning itself) are expected
# — commit them with a governance allowlist comment in the same commit
# External matches: remove, redact, or move to gitignored files before committing
```

**What counts as a self-referential match:** a file that documents the scanning procedure itself will contain the patterns it is scanning for. These are false positives. The test is: does this string, at this location, expose or reference real secret material? If not, add an allowlist comment (`# gitleaks:allow` or your scanner's equivalent) in the same commit.

**Never commit if there is any match you cannot explain.** When in doubt, ask.

### Step 4 — Audit trail check

Background processes (auto-push jobs, CI bots) can scoop your writes under their own commit subject. Before adding and committing:

```bash
# Check the last two commits
git log -2 --oneline

# Check what's actually staged vs. what you think is staged
git diff --cached --stat

# If the last commit is NOT what you expected (wrong subject, wrong author, unexpected files):
# — check who committed it (git log -1 --format="%an %ae")
# — check when (git log -1 --format="%ai")
# — if an automated process scooped your work, verify the content is correct before adding more
```

If an automated commit has run since your last manual commit, verify its content is correct before proceeding. Do not add new changes on top of an audit-trail anomaly without understanding it.

### Step 5 — Commit

```bash
git add [specific files — never git add -A without a full diff review]
git commit -m "Session YYYY-MM-DD: [headline from changelog entry]"
```

Commit message rules:
- First line: session date and headline (matches the changelog entry).
- If any allowlist comments were added in step 3, note them: "allowlisted: [N] governance self-refs".
- Never `git add -A` without first reviewing `git diff --stat` to confirm no unexpected files are staged.

### Step 6 — Next-session contract

Before closing, write the contract for the next session. This is the forward half of the loop: the contract's acceptance criteria become the next session's goals, and the next close's **Contract score** (step 1 footer) is scored against them. Contract → execute → score → new contract.

For each candidate task:

```markdown
### Task: [name]  (picked because: [one line — unblocker / highest risk / user-facing / oldest debt])
- Goal: [one sentence]
- AC: [binary, verifiable: "build passes with 0 errors", "PR #N open containing X", "file Y exists and contains Z" — never "improve A" or "make progress on B"]
- Fits one session: yes — can be done AND verified in ≤90 minutes
```

Rules:

- **Scope gate:** if a task cannot plausibly be completed AND verified within roughly 90 minutes of focused work, it does not enter the contract. Split it into a smaller slice whose AC fits, or defer it. Every task in the contract must be finishable and checkable inside one session. (Token budget can serve as a soft secondary signal if you track it, but wall-clock is the legible test.)
- **Binary AC only.** Each criterion must be checkable by a human or a command with a yes/no answer. If verification requires resources the session won't have (credentials, a deploy, another person), rewrite the AC until it is verifiable in-session — or the task doesn't go in.
- **No cherry-picking.** The contract must be drawn from the highest-priority outstanding work, and each task carries a one-line "picked because". An agent that fills the contract with easy wins to score well is gaming the loop. Watch also for AC-granularity gaming: trivially narrow criteria ("file exists") inside genuinely hard tasks.
- **Amendment escape.** If priorities legitimately change mid-session, the next session may amend the contract — with a one-line logged reason. Amendments are scored separately from the original contract: a justified pivot is healthy; silent drift is not. Do not complete stale contract tasks just to keep a clean score ("zombie work").
- **The human verifies at open.** The next session begins by checking the previous contract's AC — the human (or at minimum a fresh agent context) confirms them, not the agent that did the work grading its own homework at close.

Put the contract in the session-starter handoff (step 7) if one is generated, otherwise at the bottom of the status snapshot.

### Step 7 — Session-starter handoff (if near context limit)

If this session is ending because the context window is near its limit, or if the next session will need to reconstruct complex state:

Generate a compact handoff file (under 2K tokens) that the next session reads FIRST, before the changelog, before the status snapshot. Structure:

```markdown
# Session Starter — YYYY-MM-DD [track]

## Read this first
[2-3 sentences: where we are, what the next session must NOT do before reading this]

## Exact task state
[What was being worked on at close, step-by-step state — specific enough to resume without re-deriving]

## Decisions made this session (not yet in ledger)
- [Decision 1 — add to ledger at start of next session]

## Open questions (not blockers, but unresolved)
- [Question 1]

## Next-session contract (from step 6)
[Paste the contract here — the next session's first action after reading is to verify the PREVIOUS contract's AC, then work this one]

## First action for next session
[Imperative: exactly what to do first, no ambiguity]
```

Save to a known location (e.g., `OUTPUTS/YYYY-MM-DD-SessionStarter-TRACK.md`). Tell the human where it is.

---

## Why each step exists (failure modes)

| Skipped step | What happened | Cost |
|-------------|--------------|------|
| Status snapshot | Next session spent 15-20 minutes re-deriving system state from scratch | Wasted session budget |
| Audit trail check | Automated commit scooped a file change; next session thought the change was missing and reapplied it as a duplicate | Merge conflict and confusion |
| Forbidden-string scan | An employer name in a comment nearly passed into a public scan | Near-miss; caught by code review |
| Specific git add | `git add -A` staged a credentials file that was supposed to be gitignored | Credential in history |
| Session-starter handoff | Context-limit close left the next session reconstructing state for 30 minutes from changelog archaeology | Avoidable overhead |
| Economics footer | Token burn crept up 3x over a month of sessions with nobody noticing; no baseline existed to compare against | Silent cost bloat |
| Next-session contract | Sessions repeatedly picked up tasks too large to finish, leaving half-done work and context-limit scrambles | Rework and broken handoffs |

---

## Checklist (quick reference)

- [ ] Changelog entry appended (specific, honest verification state, NEEDS-OWNER surfaced)
- [ ] Economics footer written (MEASURED/EST never mixed; contract scored against PREVIOUS session's contract; anomalies one line)
- [ ] Status snapshot updated in place (under 5K tokens; archived if over)
- [ ] Forbidden-string scan clean (self-referential matches explained and allowlisted)
- [ ] Audit trail checked (`git log -2` confirms no unexpected automated commits)
- [ ] Staged diff reviewed (`git diff --cached --stat` confirms only intended files)
- [ ] Committed with descriptive subject line
- [ ] Next-session contract written (binary AC, scope gate applied, "picked because" on each task)
- [ ] Session-starter handoff generated if near context limit
- [ ] Human notified of any NEEDS-OWNER items before session ends

**Time budget: the entire ritual, including the two new steps, must add no more than ~5 minutes over the original 10-15. If it runs long, truncate the economics footer and contract to essentials — a ritual that takes too long gets skipped, and it gets skipped on exactly the bad days when it matters most.**

---

## What this skill does NOT do

- Does not push to remote — push is a human gate unless you have an auto-push policy and have explicitly delegated it.
- Does not make deployment decisions — those are surfaced as NEEDS-OWNER items.
- Does not run tests — it checks verification state and reports what was run; if tests weren't run, it says so.
- Does not do billing-grade cost accounting — the economics footer detects trends, not invoices. Expect the first 3-5 sessions of estimates to be badly calibrated; that's what the weekly calibration ratio is for.
- Does not enforce output style during the session — pair with a terse-output skill (e.g., `chatterbox` in this repo) and use the economics footer as the trigger for turning it on.
