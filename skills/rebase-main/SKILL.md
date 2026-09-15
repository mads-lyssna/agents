---
name: rebase-main
description: Rebase the current branch onto the latest origin/main, resolve straightforward conflicts, ask the user about conflicts with semantic ambiguity, and request explicit confirmation before force-pushing with lease. Use when the user asks to update, sync, or rebase their branch onto main.
---

# Rebase onto main

Update the current branch from `origin/main`, preserving the intent of both the branch and the newer mainline changes.

## 1. Check the branch

- Confirm the repository is not in the middle of another Git operation.
- Identify the current branch and its upstream. Do not proceed from a detached HEAD.
- Require a clean worktree and index. If there are uncommitted changes, stop and ask the user how they want to handle them; do not stash or discard them automatically.
- If the current branch is `main`, stop and tell the user rather than rebasing it onto itself.

## 2. Fetch and rebase

Run:

```bash
git fetch origin main
git rebase origin/main
```

If the fetch or rebase fails for a reason other than merge conflicts, stop and report the error. Do not guess at authentication, repository, or Git configuration fixes.

## 3. Resolve conflicts

For each conflict:

1. Read the conflicted file, its staged base/ours/theirs versions when useful, and the commit currently being replayed (`git rebase --show-current-patch`). Understand the intent of both sides before editing.
2. Resolve straightforward conflicts directly when the intended combined result is clear, such as independent additions, moved unchanged code, or formatting around an unambiguous change.
3. If resolving the conflict requires choosing between valid behaviours, changing an API or data contract, discarding meaningful logic, or otherwise guessing at intent, leave the rebase paused and ask the user a focused question. Explain the competing interpretations and identify the affected file. Continue only after they answer.
4. Remove conflict markers, stage the resolved files, and continue with `GIT_EDITOR=true git rebase --continue`.
5. Repeat until the rebase completes. Never use `git rebase --skip` or abort the rebase without the user's approval.

After conflict resolution, inspect the resulting diff and run any narrow, readily available validation needed to catch mistakes introduced by the resolution. If validation exposes semantic ambiguity, ask the user rather than inventing a fix.

## 4. Confirm before pushing

When the rebase is complete:

- Confirm the worktree is clean and summarize the rebase, conflict resolutions, and validation performed.
- Identify the configured upstream push target. If there is no upstream or the target is ambiguous, ask the user which remote branch to use.
- Show the exact `git push --force-with-lease ...` command you propose to run and ask for explicit confirmation.
- Do not treat the original rebase request as push approval. Wait for a new affirmative response given after showing the command.

After confirmation, run only the approved force-with-lease push. If the lease rejects the push, do not retry with plain `--force`; report that the remote changed and ask the user how to proceed.
