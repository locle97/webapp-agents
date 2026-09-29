# webapp-agents

A Claude Code plugin with agents and skills that work on any web app project. Nothing in it is tied to one app:
each project describes itself in its own `CLAUDE.md`, and the agents read it at the start of every task.

## What's inside

| Component | Kind | Job |
|---|---|---|
| `orchestrator` | agent | Grills the user, writes `docs/missions/<mission>/intent.md` with a Contract both teams accept, then loops techlead → QA → fix until every acceptance criterion passes |
| `playwright-site-explorer` | agent | Maps the site: writes `docs/sitemap/` and one seed per area |
| `playwright-test-planner` | agent | Writes `specs/<feature>.plan.md`, scoped by an effort level |
| `playwright-test-generator` | agent | Turns one plan scenario into one test file and runs it |
| `playwright-test-healer` | agent | Fixes failing tests, or marks them `test.fixme()` |
| `playwright-qa-manager` | agent | Runs planner → generator → healer and tracks state in `docs/qa-missions/`, or in `<mission>/qa/` under the orchestrator |
| `techlead`, `feature-designer`, `feature-builder` | agents | Design → Build pipeline for one cycle of a mission (`<mission>/build/spec.md`, `plan.md`), plus defect fixes |
| `playwright-cli` | skill | Browser automation reference. `references/` holds the test-generation workflow, effort levels and project conventions |
| `qa-pipeline` | skill | `/webapp-agents:qa-pipeline [low\|medium\|high] <mission>` starts the QA manager |
| `grilling`, `grill-me` | skills | Interview skills (vendored from [mattpocock/skills](https://github.com/mattpocock/skills)), used by `orchestrator` |

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

Agents are plugin-scoped: spawn them as `webapp-agents:<agent>`. `claude --agent orchestrator` and
`claude --agent playwright-qa-manager` work as long as no other plugin uses the same name.

## Missions: the orchestrator

`claude --agent orchestrator`, then describe what you want. The orchestrator runs the whole thing:

```
INTENT → CONTRACT_REVIEW → APPROVAL → per cycle: BUILD → VERIFY → TRIAGE ─┬─ all ACs pass → next cycle
(grill)  (techlead + QA)   (you)      (techlead) (QA)                     └─ defects → FIX (techlead) → VERIFY
                                                                   → FINAL_VERIFY → finish (PR / merge / keep)
```

- **One human gate.** You approve the intent, its contract and the autonomy settings (branch, finish action, QA
  data safety, effort, fix-round budget). After that the loop runs on its own and only comes back to you for a change
  to what the contract promises, an action you didn't pre-approve, an exhausted budget, or a real blocker.
- **The contract.** A section of `intent.md` with acceptance criteria (`AC-n`, Given/When/Then), the UI surface
  (routes, roles, accessible names, `data-testid`s, exact messages), the API surface, data and environment. The
  techlead and the QA manager both review and accept it before you see it. The build team implements exactly it, the
  QA team tests exactly it, and a team that thinks it is wrong raises a Contract Change Request instead of working
  around it. When a test fails, the contract says who is wrong: a defect goes back to the techlead, a gap becomes a
  contract change.
- **One docs folder per mission:**

  ```
  docs/missions/<mission>/
    intent.md  status.md                  orchestrator
    build/     status.md, spec.md, plan.md, cycles/<NN>/fixes/   techlead team
    qa/        mission.md, test-plan.md   QA team
  ```

  Test code stays in `tests/`. `docs/missions/README.md` indexes every mission.

Grilling now lives in the orchestrator: the techlead starts only from an approved mission folder. `/qa-pipeline`
still runs stand-alone QA missions in `docs/qa-missions/`.

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
