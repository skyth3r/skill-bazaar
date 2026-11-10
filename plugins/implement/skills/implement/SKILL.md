---
name: implement
description: Implement a Jira ticket, a Linear issue, a markdown plan file, or a plan written in the prompt end to end - move the ticket to In Progress, scope it, build it on a worktree branch, simplify, run review rounds, ship through ship-pr. Use when the user gives a Jira key or URL, a Linear issue ID or URL, a path to a plan `.md`, or an inline plan to implement or start.
argument-hint: "[--jira|--linear] <issue key or URL, plan .md path, or inline plan>"
---

# Implement a ticket or plan

Invoking this skill authorises moving a Jira ticket or Linear issue to In Progress. Pushing, opening a PR, and any check or change against production wait for the user's go-ahead.

This workflow supports Claude Code and Codex. Select the execution paths below from the current host and its available tools; never infer the host from which plugins happen to be installed.

Where the repo has `docs/agents/issue-tracker.md`, its tracker, team, project, and state names win over the defaults here.

## 1. Load the source

Take the source from the arguments or the message; with none, ask for it. Pick the first that fits:

- **Tracker issue**: a `--jira` or `--linear` flag, a Jira URL (`*.atlassian.net`), a Linear URL (`linear.app`), or a bare key (`ABC-1234`). A bare key with no flag takes its tracker from `docs/agents/issue-tracker.md`, else from whichever single tracker MCP server is connected; both connected, ask.
- **Plan file**: a path ending `.md` that exists. Read it whole, plus any file it links to that the plan depends on.
- **Inline plan**: otherwise, the plan is the prompt text itself.

Tool prefixes and schemas vary by host: discover the available tracker tools and inspect their parameters before calling them. Tracker MCP unavailable or unauthenticated: say so, ask the user to paste the issue, and report the status moves as not done.

### Jira

Through the Atlassian MCP server:

- `getAccessibleAtlassianResources` once for the `cloudId`.
- `getJiraIssue` with `view: "evidence"` (includes issue links), then `listJiraIssueComments`. Read a linked issue only where the description leans on it. Operations not exposed as direct tools go through `discover`.

### Linear

- `get_issue` with `includeRelations: true`, then `list_comments`. Read a related issue only where the description leans on it.

### Status

- State already In Review or Done: report it and ask before continuing.
- Move the issue to In Progress unless it is there already:
  - **Jira**: `getTransitionsForJiraIssue`, then `transitionJiraIssue` with the transition whose target status is In Progress. No such transition: report the workflow's available transitions and continue.
  - **Linear**: `save_issue` with `state: "In Progress"`.

## 2. Scope

Done when every open question is answered and the user has approved the plan.

- **Open decisions** belong to the user: options to pick between ("Decide one", A/B), or unresolved questions. Ask 1-3 targeted questions, offering the source's recommendation as the suggested default.
- **Acceptance criteria** missing from the source: derive them, state them, and have the user confirm.
- **Decisions in the invoking message** override the source text. Fold the scope they imply (dead code to delete, docs to realign) into the plan.
- **Plan**: each acceptance criterion paired with the change that meets it, what stays untouched, and the blast radius the source names (migrations, production data, deploy config).
- **Blast-radius unknowns** (row counts, deploy hooks, whether a migration auto-applies): check read-only once the user allows it; when access fails, hand the user the exact place to look and carry on building.

## 3. Build

Done when every acceptance criterion maps to a change or a stated reason it needs none, and the repo's definition-of-done checks pass.

- Read all applicable `AGENTS.md` and `CLAUDE.md` instructions, including referenced global guidance. Follow the current host's instruction precedence and the repo's branch, worktree, and setup conventions.
- Inspect the current checkout before creating a worktree. Reuse it when it is already the isolated worktree for this work. In a managed checkout where branch or worktree creation is unavailable, report the limitation and use the host's supported handoff rather than bypassing it.
- Resolve the review base from `origin/HEAD`, then `main`, `master`, or `trunk`; ask only if detection fails. Pin `git merge-base <base> HEAD` as the **fixed point** for simplification and every review round.
- Commit on the branch.

## 4. Simplify

Done when the agent's pass is applied or rejected and the definition-of-done checks pass again.

- **Claude Code**: the `code-simplifier:code-simplifier` agent.
- **Codex**: dispatch a native subagent to simplify the branch's changes.
- Host cannot dispatch the pass (on Claude Code, the plugin installs with `/plugin install code-simplifier@claude-plugins-official`): report simplification as not run, naming the fix for the user, and continue to step 5.

Give the simplifier the worktree path, the `git diff <fixed point>...HEAD` range, the acceptance criteria, and the repo conventions it must hold to (`CLAUDE.md`, `AGENTS.md`, `CONTEXT.md`). Tell it to preserve behaviour and to leave tests alone unless a test is itself the duplication.

- **Behaviour changes are out of scope.** Review the agent's diff. Reject anything that alters what the code does, or that a documented convention contradicts, keeping the reason for step 6.
- Re-run the repo's definition-of-done checks, then commit any applied simplification separately so the review covers it too.

## 5. Review loop

Done when a completed review lists no findings, or after three completed rounds.

Each round runs in the worktree with all branch changes committed and the working tree clean, and reviews `git diff <fixed point>...HEAD`. An empty diff is not a completed review: report it in step 6.

### Claude Code

Needs the `codex` plugin (`/plugin install codex@openai-codex`). Missing: report review as not run, naming the install command, and continue to step 6.

```bash
CODEX=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/ | sort -V | tail -1)
node "$CODEX/scripts/codex-companion.mjs" review --wait \
  --cwd "<worktree>" --scope branch --base "<fixed point>"
```

- `--scope branch` reviews the committed branch; omitting it reviews the working tree whenever the tree is dirty. `--base` keeps the pinned comparison across rounds.
- `/codex:review` sets `disable-model-invocation`, so the companion script is the only way in from here. Skip `--json`: it buries the review under a `codex` key and flattens a real error into `"Reviewer failed to output a response."`
- **Two output shapes, so read the text rather than matching a string.** The app-server runtime prints `# Codex Review`, a summary, then `Full review comments:` with a `- [P1] <title> - <path>:<lines>` block per finding. The direct-CLI fallback prints `Verdict: approve|needs-attention` and `No material findings.` Either way the question is the same: does it list findings?
- On failure, `node "$CODEX/scripts/codex-companion.mjs" status` gives the job's phase and log, `setup` names what is wrong. Report any setup or authentication action needed; configuration changes require separate authorization.

### Codex

Invoke the `code-review` skill. Supply the fixed point, the loaded issue or plan and approved scope as the spec, and the worktree path. Its Standards and Spec reviews run in independent native subagents.

- Both axes must complete for a round to count. Preserve their separate reports.
- Missing skill or subagent capability: report review as not run and continue to step 6.

### Failed or incomplete rounds

A failed round counts as nothing and never establishes a clean review. Report the cause and continue to step 6 with the review marked not run or incomplete, retaining completed rounds and any findings already returned. `Reviewer failed to output a response.` covers quota exhaustion, an expired login and a dead runtime alike; diagnose before claiming a cause.

### Acting on findings

- **Triage every finding** against the acceptance criteria and the repo's conventions (`CLAUDE.md`, `AGENTS.md`, `CONTEXT.md`, `docs/adr/`). Apply the ones that hold. A finding that contradicts a documented convention loses to the convention; keep the reason for every one you reject.
- Fix, re-run the repo's definition-of-done checks, commit, then run the next round.
- Findings still open after completed round three: stop and give the user each one with your read on it.

## 6. Record and hand back

- Report to the user: the decisions taken, what changed, checks run, the simplification pass, the review method and completed rounds, each finding rejected with its reason, anything left unverified. Leave a plan file untouched.
- On approval: invoke the `ship-pr` skill (in this marketplace), passing the tracker and issue key where there is one. It runs the local checks, pushes, opens the PR, watches CI to green, and sets the issue to In Review with the PR attached. `post-merge` picks up after the merge.
