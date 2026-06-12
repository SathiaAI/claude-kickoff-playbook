# key-leak-triage — Triage Before Revoke

**Origin:** extracted from 100+ AI working sessions building a production consumer AI product — a solo founder working with an AI workforce.

---

## The problem

The instinct when you discover a leaked API key is to revoke it immediately. That instinct is wrong about half the time, and when it's wrong, you've turned a security incident into a production outage.

The correct sequence is: understand who uses the key before you pull the plug on it.

## The failure that produced this

During a council runner build, a gateway key ended up in the runner script itself — committed to a private repo, but still in version history. The first instinct: revoke now, rotate later.

Instead: ran the triage procedure. Grepped the codebase and deployment platform for the key and its name. Result: the key existed only in that one runner script, on one developer machine, with zero production consumers. Classification A — local-only. Rotation took five minutes: provision new key, update the script, revoke the old one. No incident.

The procedure's other branch matters just as much. When triage classifies a key as runtime-critical — consumed by production services or deployment-gating CI — the rotate-then-revoke path is what stands between you and a self-inflicted outage: provision the replacement, deploy it to every consumer, verify health, and only then revoke. The consumer-discovery step is what tells you which branch you're on, and it routinely contradicts assumptions.

The same failure path — skipping consumer discovery — also revealed a subtler trap: UI labels on keys in provider dashboards are not reliable. A key labeled "local dev only" was found, via grep, to be set as an environment variable in a production service. Always grep the actual configs.

## What the skill does

1. Forces a freeze before any revoke.
2. Enumerates every consumer via grep across code, CI, and deployment configs.
3. Classifies the key as local-only (safe to revoke immediately) or runtime-critical (rotate-then-revoke).
4. Walks through abuse-log review across the exposure window.
5. Verifies the old key is dead and services are still healthy after revoke.
6. Closes the leak path: gitignore patterns, secret scanning in CI, pre-push hooks, and — critically — retagging after history rewrite so stale tags don't re-expose scrubbed commits.

## Key insight

The discipline is counterintuitive but simple: the 10 minutes you spend on triage is insurance against the hours you spend recovering from a self-inflicted outage. Triage first. Revoke only when you know what you're cutting off.
