# public-repo-scrub — Pre-Publish Scrub for Private-to-Public Moves

**Origin:** extracted from 100+ AI working sessions building a production consumer AI product — a solo founder working with an AI workforce.

---

## The problem

Content drafted in a private workspace accumulates internal context that is invisible when you're inside it: employer names, real pricing from planning docs, internal issue IDs, specific session labels, infrastructure hostnames. When that content is moved to a public repository, those references go with it unless explicitly hunted.

The second problem is subtler: git history rewrites that clean up a leaked secret don't automatically clean up the tags that point to the pre-rewrite commits. A tag created before the rewrite still exposes the scrubbed history to anyone who fetches it.

## The failures that produced this

Before one public push, an independent grep of the entire file tree — not just the files being actively worked on — caught internal references that drafting had missed, including real pricing in an example config block and war stories that made specific claims drawn from internal session data rather than verifiable facts. Neither would have been caught by reviewing only the authored files.

Separately, after a git history rewrite to remove a leaked secret, the repository still had tags pointing at the pre-rewrite commits. The tags had not been updated. Anyone fetching the tag would receive the old history, including the content that had been removed. Auditing and recreating all tags from the new HEAD resolved it.

## What the skill does

1. Provides a multi-pass grep procedure covering the entire file tree — internal names, pricing patterns, email addresses, and API key formats as separate passes.
2. Adds a claim-softening review step that reads war stories as an external reader would, catching overclaims and internal-only references in prose.
3. Verifies pushed content via sha256sum comparison between local and remote, catching partial-push mismatches.
4. Covers the full tag audit and recreate procedure required after every git history rewrite.

## Key insight

The files you authored are not the only files at risk. Grep everything. The grep takes two minutes; the alternative is discovering what you missed after a third-party indexer has cached it.
