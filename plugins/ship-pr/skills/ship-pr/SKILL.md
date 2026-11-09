---
name: ship-pr
description: Run the local checks, push a branch to origin, open the PR with gh, then watch CI to green - a red run gets diagnosed, fixed, pushed and re-watched, capped at three rounds. Use when the user asks to push and open a PR, or to watch, chase or fix CI on a branch or PR.
---

# Ship a branch through CI

Invoking this skill authorises the push and the PR. It authorises nothing else: never `--force`, never `--no-verify`, never a push to `main`, never a change to the workflow files to make a check pass.

Run every command from the branch's worktree. Read all applicable `AGENTS.md` and `CLAUDE.md` instructions. Branch, PR and check conventions follow the current host's instruction precedence.

## 1. Check locally

CI is the slow way to learn what a local run says in a minute. Nothing is pushed until this passes.

- **The checks are the repo's own**, from its definition of done: the lint, test and build commands it names, in the order it names them, plus any it makes conditional on the files this branch touched (browser journeys, database-backed tests). Read them there rather than guessing a package manager.
- No definition of done written down: fall back to the CI workflow's steps, and say which file you took them from.
- A conditional check the diff does not trigger is skipped on purpose - name it and the reason in the report, never silently.
- **CI lints more than code.** Run the repo's commit-message and PR-title checks too, even though no package script covers them. A subject that is four characters too long fails a four-second job after the suite has already passed, and fixing it once pushed means rewriting history.

```sh
base=$(git merge-base origin/HEAD HEAD)
pattern='^(build|chore|ci|docs|feat|fix|perf|refactor|revert|style|test)(\([^()]+\))?!?: .+'
git log --format='%s' "$base"..HEAD | while IFS= read -r s; do
  printf '%s' "$s" | grep -Eq "$pattern" || echo "not Conventional Commits: $s"
  [ "${#s}" -lt 72 ] || echo "${#s} chars, limit 71: $s"
done
```

  Take the pattern and the limit from the repo's own workflow where it lints them, so the local run and CI agree; otherwise from `CLAUDE.md` or `AGENTS.md`. Apply the same two rules to the PR title you write in step 3.
- **Red here is a stop, not a round.** Fix it, re-run the checks, and only then push. A check that fails for an environment reason (a service not running, a missing local secret) gets said out loud before the push, not waved through.

## 2. Push

- Target is the argument's branch, else the current one. `main`, `master` or `trunk` is a stop.
- `git status --porcelain` must be empty. Uncommitted work is a stop: report it and ask.
- `git push -u origin <branch>`.
- A pre-push hook that blocks (gitleaks, a linter) is a stop. Fix what it found; a secret in the history needs the user.

## 3. Open the PR

- `gh pr view <branch> --json url,state` first. An open PR already exists: reuse it, go to step 4.
- Otherwise `gh pr create --base <default branch> --title "<conventional commit subject>" --body "<what changed and why>"`.
- The body is short: the change, the reason, the checks run in step 1. No session links, no agent trailers, no secrets, no values pulled out of `.env`.

## 4. Watch

```sh
gh pr checks <branch> --watch --fail-fast --interval 30
```

Exit 0 is green, and the skill is done at step 6. Non-zero is **red** and starts a round.

The command outlives the tool timeout on a slow pipeline; re-running it just re-attaches, so re-run rather than assuming a verdict. "No checks reported" on a fresh PR means CI has not registered yet: re-run once after a short wait, and if it stays empty say so instead of calling it green.

## 5. Round: diagnose, fix, push

Three rounds at most. A round is one red run turned into one pushed fix.

- **Read the failure.** `gh run list --branch <branch> --limit 1 --json databaseId,conclusion,workflowName`, then `gh run view <id> --log-failed`. The failing step and its first error, not the summary line.
- **Reproduce locally** before editing. Step 1 passed, so a red check means either a check CI runs that you did not, or a difference in the environment - find which before touching code. A fix that never went red locally is a guess, and a guess costs a round.
- **Fix the cause.** A test that fails for a real reason gets the code fixed, not the test deleted, skipped or loosened. Never retry hoping.
- Re-run step 1, commit, `git push`, back to step 4.

**Infrastructure failure** (runner died, registry timeout, network) is not a round: `gh run rerun <id> --failed` once, and if it fails the same way again treat it as red and diagnose.

**After round three**, stop. Give the user each remaining failure, its log excerpt, and your read on it. Do not start a fourth.

## 6. Report

The PR URL, the final check status, the local checks run and any skipped with the reason, and one line per round: what was red, what fixed it. Where the branch references a tracker issue, attach the PR to it and set it to In Review.
