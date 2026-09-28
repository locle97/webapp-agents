# Project conventions

The agents in this plugin work on any web app with a Playwright test suite. They don't hard-code a project's
paths, project names or login flow. They resolve them once, at the start of a task, from:

1. The project's `CLAUDE.md` (or `AGENTS.md`), especially a **Playwright agents** section if it has one (template
   below). This source wins.
2. `playwright.config.ts` (or `.js`): `testDir`, `use.baseURL`, `projects` (names, `dependencies`, `testMatch`,
   `testIgnore`, `use.storageState`).
3. The defaults in the table below.

If the live app contradicts something written here or in the project's docs, trust the app.

## Placeholders

Agent prompts and the other references use these names. Replace each one with the value you resolved.

| Placeholder | Meaning | Default |
|---|---|---|
| `<base-url>` | App under test | `use.baseURL` from the config, or a `BASE_URL` constant in the fixtures module |
| `<tests-dir>` | Test root | `testDir` from the config, else `tests` |
| `<fixtures>` | Module that exports `test`/`expect` (and often `BASE_URL`), if the project has one | `<tests-dir>/fixtures.ts` |
| `<helpers-dir>` | Shared test helpers | `<tests-dir>/utils` |
| `<base-seed>` | Base seed test | `<tests-dir>/seed.spec.ts` |
| `<seeds-dir>` | Per-area seeds (`<area>.seed.spec.ts`) | `<tests-dir>/seeds` |
| `<setup-project>` | Project that logs in and writes the storage state | the project other projects list in `dependencies`, usually `setup` |
| `<storage-state>` | Saved auth state file | `use.storageState` from the config, usually `playwright/.auth/user.json` |
| `<login-url>` | Where the app redirects when a session is missing or expired | find it from the setup file's first `goto` (for example `/login`, `/sessions/new`) |
| `<smoke-projects>` | Projects to run for `low`/`medium` verification | `--project=chromium`, plus any other project that runs files `chromium` ignores (for example a serial Chrome project for tests that share one user) |

A project with no auth has no `<setup-project>` or `<storage-state>`. In that case skip every login and
`state-load` step, and drop `--no-deps`.

The agents' own outputs always go to the same places, relative to the repo root: test plans in
`specs/<feature>.plan.md`, the sitemap in `docs/sitemap/`, and QA mission logs in `docs/qa-missions/`.

## Project rules the agents look for

Read these from the project's `CLAUDE.md` and pass them to every subagent you brief:

- **Data safety**: is the target production or real user data? What may tests create, change or delete, and how must
  they restore it? If the project says nothing, assume read-only: never create, update, delete, send or import
  anything without the user's permission.
- **Login constraints**: one-time codes (TOTP), rate limits or single-session logins mean the setup project must run
  once and never in parallel. Treat this as the default even when it isn't stated.
- **Locale and formats**: currency, dates, relative times. Use regexes or the project's helpers instead of
  hard-coded formats.
- **UI pitfalls**: dialogs that navigate away on `Escape`, auto-saving forms, controls without accessible names,
  live secrets that must never be copied.
- **Isolated projects**: tests that must run serially (they share one user's profile, for example) and the project
  that runs them.

## Template: a "Playwright agents" section for a project's CLAUDE.md

```markdown
## Playwright agents

- Base URL: https://app.example.com (from `playwright.config.ts`)
- Auth: `setup` project (`tests/auth.setup.ts`) writes `playwright/.auth/user.json`. Login page: `/login`.
  Login uses TOTP, so never run `setup` in parallel.
- Fixtures: import `test`, `expect` and `BASE_URL` from `tests/fixtures.ts`. Helpers live in `tests/utils/`.
- Smoke projects: `--project=chromium --project=settings` (`settings` runs `tests/settings` serially in Chrome).
- Data safety: production data. Read-only unless the user allows a change; tests that change data restore it in
  `finally`.
- Pitfalls: `Escape` in a dialog navigates away, so use the Cancel button. Settings forms auto-save on change.
```
