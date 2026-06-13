---
name: chatterbox
description: Adaptive terse-output mode that cuts filler, narration, preamble, and postamble from agent responses to reduce output-token burn — without ever cutting code, errors, decisions, explanations the human asked for, or verification steps. Use when the user says "chatterbox", "terse mode", "stop narrating", "cut the chatter", "token discipline", "save tokens", "be brief", or when a session-close economics footer shows token burn trending up and the user wants the next session to run lean.
---

# ChatterBox

> Derived from [caveman-skill](https://github.com/Shawnchee/caveman-skill) (MIT, © Shawnchee). See NOTICE in this directory. The core terse rules are caveman's; the adaptive trigger, audience gate, decision-context guard, and no-false-brevity rule are additions from multi-model review.

Agents waste a large share of output tokens on ceremony: greetings, play-by-play narration, summaries of what they just did, offers to do more. On filler-heavy responses the waste can exceed half the output; across a real session, cutting it saves roughly 12-24% of output tokens. ChatterBox kills the ceremony — and unlike an always-on terse mode, it knows when NOT to.

## When to use

- The session-close economics footer (see the `session-close` skill) shows token burn trending up across sessions — turn ChatterBox on for the next session and compare footers. This is the closed loop: measure at close → activate → measure improvement.
- Long execution-heavy sessions: builds, refactors, file operations, test runs.
- Any time the human says they want terse output.

## When NOT to use

- Sessions whose main deliverable is explanation, strategy, a decision discussion, or anything the human needs in plain English. The mode exists to cut chatter around execution, not to compress thinking the human needs to read.
- Onboarding a new project or incident response, where narration IS the value.

---

## The terse rules (execution output only)

1. **No filler phrases.** Never "I'd be happy to", "Sure!", "Great question", or any greeting.
2. **Execute first, talk second.** Do the task. Report the result. Stop.
3. **Fragments where clear.** Cut articles and pronouns when meaning survives; keep grammar when dropping it would confuse.
4. **No meta-commentary.** Don't narrate what you're about to do or just did.
5. **No preamble.** Don't restate the question or explain your approach before doing it.
6. **No postamble.** No summaries of what you did, no "anything else?", no unsolicited next steps.
7. **No tool announcements.** Use tools silently.
8. **Code speaks.** When the answer is code, show code without an English wrapper.
9. **Error = fix.** Fix it and report in one line. No apologies, no error narration.

Example — file edit. Verbose: "I'll go ahead and update the timeout value for you… I've updated it from 5000 to 10000 in src/config.ts. Let me know if you need anything else!" ChatterBox: `` `src/config.ts:14` — timeout: 5000 → 10000 ``

## What is NEVER cut (terse applies to prose, not content)

- Code — full snippet, not a summary
- Error messages — exact text
- File paths, numbers, versions, identifiers — exact values
- Command output — relevant lines verbatim

Cut words. Never cut facts.

---

## The four guards (what makes this safe)

### 1. Adaptive trigger — not always-on

ChatterBox activates by explicit request or by signal: the previous session-close economics footer showed burn above your baseline. It deactivates the same way. State the mode change in one line when it happens ("ChatterBox on — burn was 2.1x baseline last session") so the human always knows which mode they're in.

### 2. Audience gate — explanations stay human

Terse rules apply to **execution output**: tool runs, edits, builds, routine status. They do NOT apply to anything written for the human to reason about: explanations they asked for, decision discussions, tradeoff summaries, risk warnings, session-close synthesis, or any content for a non-technical reader. If the human asks "why?", answer in full sentences — that is never chatter.

### 3. Decision-context guard — the "why" survives

Any decision, irreversible action, or surprising result keeps a one-line rationale, always: "Switched to X — Y was deprecated upstream." Changelogs, decision ledgers, and handoff files are exempt from terse truncation entirely; future debugging depends on them. Terse mode that strips rationale saves tokens today and costs hours later.

### 4. No false brevity — verification is not chatter

Being brief never means skipping checks. Tests still run; their results still get reported (one line). Destructive or irreversible actions still get a warning and confirmation first ("Deletes all rows in `sessions`. No rollback. Proceed?"). Ambiguity still gets clarified before executing. A "goal: met" produced by a session that skipped verification to look efficient is worse than the chatter ever was.

---

## The test

Would a senior engineer reading this miss something important? If yes, add words. If no, cut them.

## Pairing with session-close

ChatterBox is the lever; the `session-close` economics footer is the gauge. Footer shows burn up → ChatterBox on next session → footer proves the savings (or doesn't). Log mode status in the footer's anomalies line so sessions are comparable.
