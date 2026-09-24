---
name: rebase-main
description: Rebase the current branch or its GitHub PR Stack onto the latest origin/main, resolve straightforward conflicts, ask about semantic ambiguity, and request explicit confirmation before pushing rewritten branches. Use when the user asks to update, sync, or rebase their branch or stack onto main.
---

# Rebase onto main

Update the current branch from `origin/main`, preserving the intent of both the branch and the newer mainline changes. For GitHub PR Stacks, update the whole dependent chain instead of rebasing one branch in isolation. Read `../github/SKILL.md` when inspecting PRs or preparing a GitHub write; its approval rules also apply to stack pushes.

## 1. Check the branch

- Confirm the repository is not in the middle of another Git operation.
- Identify the current branch and its upstream. Do not proceed from a detached HEAD.
- Require a clean worktree and index. If there are uncommitted changes, stop and ask the user how they want to handle them; do not stash or discard them automatically.
- If the current branch is `main`, stop and tell the user rather than rebasing it onto itself.

## 2. Identify the rebase path

Before running `git rebase`, check whether the branch belongs to a GitHub PR Stack:

- Check `gh extension list` for `github/gh-stack`. If installed, run `gh stack view --json` (not bare `view`, which may open a TUI). A successful result shows the locally tracked stack, its trunk, and every branch that could be rewritten. Exit code 2 means the current branch is not in a locally tracked stack; other errors need investigation, not an automatic fallback to `git rebase`.
- If no local stack is found, check for a PR with `gh pr view --json url,headRefName,baseRefName` (no PR is fine). A PR based on another feature branch, or a stack mentioned by the user, may be a remote-only stack. A PR based on `main` may still be the bottom of a stack. If stack membership is indicated but not locally tracked, stop and ask whether to check it out with `gh stack checkout <PR number or URL>` before rebasing; do not silently rebase the branch alone or initialize a new stack.
- If stack membership is known but the extension is unavailable, stop and ask the user to install `github/gh-stack` or choose a different approach. Do not install it or rewrite one layer as a workaround.

For a locally tracked stack, confirm its trunk is `main` and the intended fetch remote is `origin`; otherwise ask before changing the target. Run `gh stack rebase --remote origin` to fetch and cascade-rebase the stack from trunk upward. Do not use `gh stack sync` here: it also pushes branches and updates GitHub state. If there is no indication of a stack, use the standalone path:

```bash
git fetch origin main
git rebase origin/main
```

If the fetch or rebase fails for a reason other than conflicts, stop and report the error. Do not guess at authentication, repository, or Git configuration fixes.

## 3. Resolve conflicts

For each conflict:

1. Read the conflicted file, its staged base/ours/theirs versions when useful, and the commit currently being replayed (`git rebase --show-current-patch`). Understand the intent of both sides before editing.
2. Resolve straightforward conflicts directly when the intended combined result is clear, such as independent additions, moved unchanged code, or formatting around an unambiguous change.
3. If resolving the conflict requires choosing between valid behaviours, changing an API or data contract, discarding meaningful logic, or otherwise guessing at intent, leave the rebase paused and ask the user a focused question. Explain the competing interpretations and identify the affected file. Continue only after they answer.
4. Remove conflict markers and stage the resolved files. For a stack, continue with `gh stack rebase --continue` (which can proceed to the next branch); for a standalone branch, use `GIT_EDITOR=true git rebase --continue`.
5. Repeat until the rebase completes. Never skip commits or abort the rebase without the user's approval. For a stack, `gh stack rebase --abort` restores the whole stack; do not substitute plain `git rebase --abort`.

After conflict resolution, inspect the resulting diff and run any narrow, readily available validation needed to catch mistakes introduced by the resolution. If validation exposes semantic ambiguity, ask the user rather than inventing a fix.

## 4. Confirm before pushing

When the rebase is complete:

- Confirm the worktree is clean and summarize the rebase, conflict resolutions, and validation performed.
- For a standalone branch, identify the configured upstream push target. If there is no upstream or the target is ambiguous, ask which remote branch to use. Show the exact `git push --force-with-lease ...` command.
- For a stack, inspect `gh stack view --json` again. Name the active (non-merged, non-queued) branches and remote that `gh stack push --remote origin` would update. Explain that it pushes all those branches with per-branch leases and is not atomic; do not use `gh stack submit` or `gh stack sync` as a push substitute, since those can also change PR or stack state.
- Ask for explicit confirmation of the exact command and its targets. Do not treat the original rebase request as push approval. Wait for a new affirmative response given after showing the command.

After confirmation, run only the approved push. If a lease rejects a branch, do not retry with plain `--force`; report which branches updated, which did not, and ask how to proceed.
