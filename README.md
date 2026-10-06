# pr-feedback-verification

A `/pr-feedback-verification` skill for Claude Code and Codex that answers one
question: **was the review feedback on this PR actually addressed, and is it
ready to merge now?**

You left a review, the author pushed, and now you need to know whether your
comments landed. Run `/pr-feedback-verification <PR>`. The agent pulls every
prior review comment with the commit it was anchored to, diffs that location
against the current head, and classifies each comment as addressed, partial,
ignored, or N/A. Then it checks CI, open threads, branch freshness, and the
review decision, and gives one verdict: ready to merge, needs more work, or
blocked.

## Install

```bash
git clone https://github.com/jensvanbellen/pr-feedback-verification.git && cd pr-feedback-verification && ./install.sh
```

This symlinks the skill into both tools:

| Tool | Location | Invoke |
|------|----------|--------|
| Claude Code | `~/.claude/skills/pr-feedback-verification` | `/pr-feedback-verification` |
| Codex | `~/.codex/skills/pr-feedback-verification` + `~/.codex/prompts/pr-feedback-verification.md` | `/pr-feedback-verification` |

Both tools read the exact same `skills/pr-feedback-verification/SKILL.md`: one
source of truth, no drift. Because they are symlinks, edits to this repo take
effect immediately. `./install.sh --uninstall` removes everything. Restart open
sessions after installing so the skill is picked up.

## Requirements

- `gh` CLI, authenticated with read access to the repo.
- A local clone of the repo, so the agent can diff the flagged code between
  commits.

## Usage

| Command | What happens |
|---------|--------------|
| `/pr-feedback-verification 123` | Reconciles all review feedback on PR 123 in the current repo |
| `/pr-feedback-verification <PR URL>` | Same, for any repo |
| "did they address my review on 123?" | Scopes the reconciliation to your own comments |

Example report:

```
| Author | File:line            | Status    | Note                              |
|--------|----------------------|-----------|-----------------------------------|
| alice  | models/orders.sql:42 | Addressed | join moved to the CTE as asked    |
| alice  | models/orders.yml:10 | Ignored   | unique test still missing         |

Merge-readiness: CI ✅  threads ❌ (1 open)  branch ✅  decision ❌ CHANGES_REQUESTED
Verdict: Needs more work (unique test on orders.id)
```

## Design decisions

- **Reconcile, don't re-review.** It checks prior comments against the current
  head. For a fresh review, use a code-review skill or
  [second-opinion](https://github.com/jensvanbellen/second-opinion).
- **Anchored to the commit each comment was made on.** Status comes from
  diffing `original_commit_id` against the head SHA, not from thread state.
- **Resolved is not the same as fixed.** A resolved thread whose code never
  changed is flagged, not trusted.
- **Read-only by default.** It reports a verdict and posts nothing. On explicit
  request it drafts exactly one review, shows it to you, and posts only after
  your go-ahead.
- **Symlink install, one skill file.** Same pattern as
  [agent-handoff](https://github.com/jensvanbellen/agent-handoff): both tools
  read the identical SKILL.md, and the Codex prompt is a thin pointer so the
  slash command works there too.

## Repo layout

```
skills/pr-feedback-verification/SKILL.md    # the skill, single source of truth
codex/prompts/pr-feedback-verification.md   # thin Codex slash-command pointer
install.sh                                  # symlinks into ~/.claude and ~/.codex
```

## License

Licensed under the [MIT License](LICENSE).
