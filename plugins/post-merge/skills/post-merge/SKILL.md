---
name: post-merge
description: Clean up after a merged PR - update local main, remove the branch's worktree, delete the branch. Use when the user says a PR is merged, or asks to clean up merged worktrees or branches.
argument-hint: "[PR number or branch]"
---

# Post-merge cleanup

Invoking this skill authorises every step below, including deleting the remote branch. Stop and report at any failed check; never reach for `--force`.

## 1. Find the targets

- **Argument given** (PR number or branch): that PR's branch.
- **No argument, session inside a worktree**: the current branch.
- **No argument, session in the main checkout**: every worktree from `git worktree list` whose PR shows `MERGED` in step 2. List them before continuing.

The main checkout is the first entry of `git worktree list --porcelain`. Run every command against it by absolute path (`git -C <main>`), since the worktree directory is about to disappear. If the session entered the worktree with `EnterWorktree`, call `ExitWorktree` first.

## 2. Verify, per target

Every check passes or that target is skipped:

- `gh pr view <branch> --json state,headRefOid` reports `state: MERGED`.
- `git -C <worktree> status --porcelain` is empty.
- `git -C <worktree> rev-parse HEAD` equals `headRefOid`. A mismatch means local commits that never reached the PR.

## 3. Update main

```sh
git -C <main> switch main
git -C <main> pull --ff-only origin main
```

A dirty main checkout or a non-fast-forward pull is a stop.

## 4. Remove, per verified target

```sh
git -C <main> worktree remove <worktree>
git -C <main> branch -D <branch>   # -D: squash merges leave the branch unmerged by ancestry; step 2 proved it landed
git -C <main> push origin --delete <branch>   # only if `git ls-remote --heads origin <branch>` still lists it
git -C <main> fetch --prune
```

## 5. Report

Main's new HEAD (`git log --oneline -1`), then one line per target: removed, or skipped with the failing check.
