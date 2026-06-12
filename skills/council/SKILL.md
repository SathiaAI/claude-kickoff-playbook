---
name: council
description: Run a multi-model ensemble review before locking any significant decision. Sends an identical decision brief to 3-4 diverse frontier models via an OpenAI-compatible gateway, synthesizes structured verdicts, and writes a RAW transcript for audit. Use when the user says "run the council", "validate this decision", "get a second opinion from all models", "multi-model review", "ensemble check", "should I lock this decision", "council vote on this", "check this with multiple models", "adversarial review before I commit", or "is this decision sound". Also invoke automatically before any architectural, security, pricing, or irreversible action if the user has configured auto-council.
---

# Council — Multi-Model Decision Validation

Before you lock an irreversible decision, get four opinions. Councils catch what single-model review misses: the dissent IS often the finding.

A gateway (e.g., LiteLLM or any OpenAI-compatible router) is convenient but not required — calling each provider's API directly works the same way; the procedure only assumes you can send the same prompt to multiple models and cap spend.

## When to use

- Architecture choices (data model, service split, auth boundary)
- Security or permissions decisions
- Pricing or contract terms that will be public
- Any action that is hard to undo (deleting data, rotating master keys, publishing under a name)
- When a single-model review feels like an echo chamber

## When NOT to use

- Routine code edits, typo fixes, style changes — council overhead is not justified
- Time-critical incidents where the action is obvious and delay costs more than the risk
- Decisions that are already locked in your ledger — the council settled it; don't re-litigate

---

## Step-by-step procedure

### 1. Write the decision brief

Produce a structured brief. Keep it under 800 tokens so every model receives the same full context.

```
## Decision Brief

**Date:** YYYY-MM-DD
**Question:** [One precise sentence: what are you choosing between?]

**Context:** [2-5 sentences: what the system does, what led here]

**Options:**
A. [Option name] — [one sentence]
B. [Option name] — [one sentence]
C. (if applicable)

**Constraints:**
- [Hard constraint 1]
- [Hard constraint 2]

**Cost of being wrong:** [What breaks, what it costs to undo]

**Requested output format per model:**
RECOMMENDED: [A/B/C]
CONFIDENCE: [HIGH / MEDIUM / LOW]
TOP RISK: [one sentence]
DISSENT: [what the other options get right that you are conceding]
```

### 2. Mint a scoped virtual key

Never use or echo your master gateway key in chat. Before the run:

1. Log into your LLM gateway admin UI.
2. Create a new virtual key scoped to only the council model aliases.
3. Set a hard dollar cap (suggested: $1-$5 depending on model count and brief length).
4. Set TTL to 24 hours.
5. Store the key in a variable: `COUNCIL_KEY=sk-REPLACE-WITH-KEY`
6. Do NOT commit runner scripts that contain this key. Add the runner output directory to `.gitignore`.

### 3. Smoke-test the gateway

Model slugs deprecate without warning. Before sending the real brief:

```bash
# For each model alias, send a trivial call
curl -s https://YOUR-GATEWAY-HOST/v1/chat/completions \
  -H "Authorization: Bearer $COUNCIL_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "MODEL-ALIAS-HERE", "messages": [{"role":"user","content":"ping"}], "max_tokens": 1}' \
  | jq '.choices[0].message.content // .error'
```

If any alias returns a model-not-found error: check the gateway's `/v1/models` list and update the alias before proceeding.

### 4. Send the brief to each model

Send the identical brief to each model. Recommended composition:

| Slot | Model type | Why |
|------|-----------|-----|
| 1 | Your primary reasoning model (e.g., Claude Opus) | Deep analysis, careful reasoning |
| 2 | A second frontier model (e.g., GPT-5) | Different training data and priors |
| 3 | A third frontier model (e.g., Gemini) | Additional independent signal |
| 4 | A fast reasoning model (e.g., DeepSeek) | Speed + different architecture |

**Timeout guidance:** set per-request timeout to 120s. If a model uses thinking/reasoning mode it can burn 5-15K tokens and take several minutes — set `max_tokens` explicitly (suggested: 4096 for brief reviews, 8192 for deep analysis) to prevent runaway cost. Avoid relying on reasoning-mode models for high-volume batch runs; they can hang.

**Do not stream to a file that gets committed.** Pipe to a temp location, then copy to your RAW transcript file after review.

### 5. Collect and structure verdicts

Extract the structured output from each model response. If a model declined to use your format, extract the equivalent fields manually. Build a synthesis table:

```
| Model   | Recommended | Confidence | Top Risk | Dissent |
|---------|-------------|------------|----------|---------|
| Model-1 | ...         | ...        | ...      | ...     |
| Model-2 | ...         | ...        | ...      | ...     |
| Model-3 | ...         | ...        | ...      | ...     |
| Model-4 | ...         | ...        | ...      | ...     |
```

### 6. Synthesize

Apply these rules in order:

- **Unanimous (4/4 or 3/4 same option):** Strong signal. Surface the top risk the majority named, and the strongest dissent even minority models raised.
- **Split (2/2 or 3-way):** The disagreement IS the finding. Surface the specific axis of disagreement to the human; do not resolve it yourself. Frame it: "Models disagree on X. Model-1 and Model-3 weight [concern A] heavily. Model-2 and Model-4 weigh [concern B]. The decision hinges on which risk you accept."
- **Any model raises a blocker:** Escalate regardless of vote distribution. A single well-reasoned blocker can outweigh a majority recommendation.

### 7. Write the RAW transcript and log the decision

1. Save all model responses verbatim to a RAW file: `council-YYYY-MM-DD-DECISION-SLUG-RAW.md`
2. Add an entry to your decision ledger referencing the RAW file path.
3. If the decision is now locked, mark it LOCKED in the ledger. If still DRAFT, note what additional information is needed.

---

## Failure modes

| Failure | Symptom | Fix |
|---------|---------|-----|
| Deprecated model slug | 404 or model-not-found on smoke test | Check `/v1/models`; update alias |
| Gateway timeout on reasoning model | Request hangs > 120s | Lower `max_tokens`; switch to non-thinking variant |
| Provider ignores prompt caching | Each call charged at full price | Budget accordingly; don't rely on cache savings with all providers |
| Council key in committed file | Secret scanner fires | Immediately rotate the key; add output dir to `.gitignore`; history-rewrite if already pushed |
| Models return identical verdicts | Looks like consensus, may be correlated training | Check if you used models from the same provider family; add a model from a different training lineage |
| Split verdict paralyzes decision | Decision-maker can't decide | Identify the single axis of disagreement; council's job is to surface it, human's job is to decide |

---

## What this skill does NOT do

- Does not make the final decision — it surfaces verdicts for the human to lock.
- Does not auto-revoke or rotate keys if the council key leaks — use the `key-leak-triage` skill.
- Does not substitute for domain expertise in legal, medical, or financial matters.
