# Skill Bazaar

[Akash's](https://akashgoswami.com) plugin marketplace for Claude Code, Codex and other agents.

## Plugins

| Plugin | What it does |
|---|---|
| `post-merge` | Clean up after a merged PR: update main, remove the worktree, delete the branch |
| `ship-pr` | Run local checks, push a branch, open the PR, then watch CI to green with capped fix rounds |

Claude Code, one step per plugin:

```
/plugin install post-merge --marketplace skyth3r/skill-bazaar
/plugin install ship-pr --marketplace skyth3r/skill-bazaar
```

## Install

**Claude Code**

```sh
claude plugin marketplace add skyth3r/skill-bazaar
claude plugin install <plugin>@skill-bazaar
```

Or in one step from a session: `/plugin install <plugin> --marketplace skyth3r/skill-bazaar`

**Codex**

```sh
codex plugin marketplace add skyth3r/skill-bazaar
```

Then run `codex`, open `/plugins` and install from Skill Bazaar.

**Other agents** (Gemini CLI, Cursor, Copilot, OpenCode, ...)

```sh
npx skills add skyth3r/skill-bazaar
```

## Adding a plugin

Plugins are skills-only so the same files work on every agent.

1. Create `plugins/<name>/skills/<skill>/SKILL.md` (frontmatter `name` must match the folder).
2. Create `plugins/<name>/plugin.json` (Codex manifest: `$schema`, `name`, `version`, `description`).
3. Add an entry to `.claude-plugin/marketplace.json` (Claude).
4. Add the same entry to `.agents/plugins/marketplace.json` (Codex, needs `policy` and `category`).
5. Run `claude plugin validate . --strict`.

CI fails if the two marketplace files list different plugin names.
