---
name: pr-feedback-verification
description: Check whether review feedback on a PR was actually addressed. Reconcile each existing review comment against the current head, mark it addressed / partial / ignored, then run the merge-readiness gate. Use when the user asks "did they address my review?", "was my feedback applied?", "is this ready to merge now?", "re-review this PR", or invokes /pr-feedback-verification. Distinct from a fresh review (use a code-review skill or /second-opinion for that).
user-invocable: true
metadata:
  author: jens
---

# PR Feedback Verification

Answer one question: **was the review feedback on this PR actually addressed, and is it
ready to merge now?** This is a reconciliation against *prior* comments, not a fresh review.
For a first-time review, use a code-review skill or `/second-opinion` instead.

Read-only by default: this skill reports a verdict. It does **not** post to GitHub unless
the user explicitly asks (see "If asked to post").

## Step 1: Resolve the PR and its head

- Resolve owner/repo + number from the URL or number given. If only a number is given,
  confirm the repo from the current working directory's remote.
- `gh pr view <n> --repo <owner/repo> --json number,headRefOid,headRefName,baseRefName,state,mergeable,reviewDecision,url,title`
- Record the current **head SHA**. Every "was it fixed?" check is relative to it.

## Step 2: Collect the existing feedback

Pull every prior comment, and the commit each was anchored to:

- **Inline review comments:** `gh api repos/<owner>/<repo>/pulls/<n>/comments --paginate`
  → for each: `path`, `line`/`original_line`, `body`, `user.login`, `original_commit_id`,
  `in_reply_to_id`, and whether the thread is resolved.
- **Review submissions (verdicts):** `gh api repos/<owner>/<repo>/pulls/<n>/reviews --paginate`
  → `user.login`, `state` (APPROVED / CHANGES_REQUESTED / COMMENTED), `body`, `submitted_at`.
- **Issue-level comments:** `gh pr view <n> --repo <owner/repo> --json comments`.

If the user scoped it ("did they address *my* comments"), filter to their login
(`gh api user --jq .login`); otherwise reconcile everyone's.

## Step 3: Reconcile each comment against current head

For each substantive comment, decide its status by comparing the code it flagged at
`original_commit_id` to the same location at the current head SHA:

- `git fetch` the PR head, then `git diff <original_commit_id>..<headRefOid> -- <path>`
  (or `gh api .../compare/<base>...<head>`) to see what changed at/after the flagged lines.
- Read the current state of the flagged code to judge whether the concern is resolved.

Classify each as:
- **Addressed**: the flagged code changed in a way that resolves the concern.
- **Partial**: touched but the concern isn't fully met.
- **Ignored**: unchanged, concern still stands.
- **N/A**: the code was removed/refactored away, or the comment was a question already
  answered in a reply.

A resolved thread with unchanged code is suspicious. Flag it, don't assume resolved.

## Step 4: Merge-readiness gate

Independently of the comments, check:
- **CI:** `gh pr checks <n> --repo <owner/repo>`. All required checks green.
- **Threads:** any unresolved review threads (from Step 2).
- **Branch freshness:** is head up to date with base? (`mergeable` / `mergeStateStatus`
  from `gh pr view --json mergeable,mergeStateStatus`; behind-base → needs a merge/rebase.)
- **Verdict state:** current `reviewDecision`.

## Step 5: Report

Give a compact table: one row per comment (author, file:line, status, one-line note), then:
- **Merge-readiness:** CI / threads / branch / decision, each ✅ or ❌ with the blocker.
- **Verdict:** `Ready to merge` / `Needs more work` (list what's left) / `Blocked` (list
  blockers).

Fully qualify any cross-repo issue/PR reference as `owner/repo#n`; a bare `#n` only for the
PR's own repo.

## If asked to post the result to GitHub

Only on explicit request. Then:
- Decide the event up front and post **exactly one** review. Submitted reviews can't be
  deleted, so if it's a gate, post a single `gh pr review <n> --request-changes` carrying
  the whole thing. Never a `--comment` pass then a `--request-changes` pass.
- Strip any agent session URL.
- End the body with, after a blank line, in italics: `*Comment created with Claude Code.*`
  (or `*Comment created with Codex.*` when Codex wrote it).
- Draft it, show the user, get the go-ahead, then post.
