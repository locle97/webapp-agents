# webapp-agents

A Claude Code plugin with agents and skills that work on any web app project. Nothing in it is tied to one app:
each project describes itself in its own `CLAUDE.md`, and the agents read it at the start of every task.

## What's inside

| Component | Kind | Job |
|---|---|---|
| `playwright-site-explorer` | agent | Maps the site: writes `docs/sitemap/` and one seed per area |
| `playwright-test-planner` | agent | Writes `specs/<feature>.plan.md`, scoped by an effort level |
| `playwright-test-generator` | agent | Turns one plan scenario into one test file and runs it |
| `playwright-test-healer` | agent | Fixes failing tests, or marks them `test.fixme()` |
| `playwright-qa-manager` | agent | Runs planner → generator → healer and tracks state in `docs/qa-missions/` |
| `techlead`, `feature-designer`, `feature-builder` | agents | Plan → Design → Build pipeline (`docs/<feature>/intent.md`, `spec.md`, `plan.md`) |
| `playwright-cli` | skill | Browser automation reference. `references/` holds the test-generation workflow, effort levels and project conventions |
| `qa-pipeline` | skill | `/webapp-agents:qa-pipeline [low\|medium\|high] <mission>` starts the QA manager |
| `grilling`, `grill-me` | skills | Interview skills (vendored from [mattpocock/skills](https://github.com/mattpocock/skills)), used by `techlead` |

## Requirements

- `playwright-cli` on `PATH` (`npm install -g @playwright/cli@latest`), and `@playwright/test` in the project.
- The `superpowers` plugin, for the techlead pipeline (`brainstorming`, `writing-plans`,
  `subagent-driven-development`).

## Install

```bash
# in Claude Code
/plugin marketplace add ~/coding/github/webapp-agents
/plugin install webapp-agents@webapp-agents
```

Agents are plugin-scoped: spawn them as `webapp-agents:<agent>`. `claude --agent techlead` and
`claude --agent playwright-qa-manager` work as long as no other plugin uses the same name.

## Setting up a project

The agents resolve paths, project names and the login flow from, in order: the project's `CLAUDE.md`,
`playwright.config.ts`, then built-in defaults (`tests/`, `tests/fixtures.ts`, `tests/utils/`, `tests/seed.spec.ts`,
`tests/seeds/`, a `setup` project writing `playwright/.auth/user.json`). The full list and a template are in
[`skills/playwright-cli/references/project-conventions.md`](skills/playwright-cli/references/project-conventions.md).

A project that follows the defaults needs nothing extra. Otherwise, or when the app has rules the agents must know
(production data, TOTP login, a serial project for shared-user tests, locale formats, UI pitfalls), add a
**Playwright agents** section to the project's `CLAUDE.md` using that template.

## Conventions every project gets

- Browsers always run headed, so you can watch.
- Read-only by default: no data is created, changed or deleted unless the user allows it.
- The setup (login) project runs once per mission. Generators use `--no-deps` and only read the storage state.
- Effort levels (`low` / `medium` / `high`) cap scenarios at 5 / 8 / 15. See
  `skills/playwright-cli/references/effort-levels.md`.
