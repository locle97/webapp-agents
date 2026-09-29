# API project conventions

The API agents resolve these once, at the start of a task, in the same order as the e2e agents (see the
`playwright-cli` skill's `references/project-conventions.md`, which also defines `<base-url>`, `<storage-state>`,
`<setup-project>`, `<login-url>`, `<fixtures>` and `<smoke-projects>`):

1. The project's `CLAUDE.md` (or `AGENTS.md`), especially an **API agents** section (template below). This wins.
2. `playwright.config.ts`: `use.baseURL`, `projects`, `testMatch` / `testIgnore`, `use.storageState`.
3. The defaults below.

If the live API contradicts something written down, trust the API, and say so in your report.

## Placeholders

| Placeholder | Meaning | Default |
|---|---|---|
| `<api-base-url>` | Origin (and prefix) that API paths are relative to | `<base-url>`. Plan and client paths include the prefix, e.g. `/api/projects` |
| `<api-dir>` | Root of the API test layers | `<tests-dir>/api` |
| `<api-fixtures>` | Module exporting `test` (with `api`, `apiAs`, `cleanup`) and `expect` | `<api-dir>/fixtures.ts` |
| `<api-project>` | Playwright project that runs `*.api.spec.ts` | a project whose `testMatch` matches `.api.spec.ts`; otherwise the first project in `<smoke-projects>` |
| `<roles>` | Role name → storage state file. `default` is always `<storage-state>`; `anon` is always no state | `default`, `anon`. More only when the project's setup writes more state files |
| `<api-auth>` | How the API authenticates | `cookie`: the storage state's cookies are enough. Or `bearer:localStorage:<key>` / `bearer:cookie:<name>`: send that value as `Authorization: Bearer` |
| `<error-shape>` | The JSON body every error returns | read it from the contract or the code; otherwise observe it (see `probing.md`) and write it to `shapes/error.ts` |
| `<openapi>` | OpenAPI/Swagger file, if the repo has one | none. Look for `openapi.*`, `swagger.*` |

Agent outputs always go here, relative to the repo root: API plans in `specs/<feature>.api.plan.md` (or the path
the QA manager gives), the API map in `docs/apimap/`.

## The API project

API tests need no browser. Ideally the config has a project just for them:

```ts
// playwright.config.ts
projects: [
  { name: 'setup', testMatch: /.*\.setup\.ts/ },
  { name: 'api', testMatch: /.*\.api\.spec\.ts/, dependencies: ['setup'],
    use: { storageState: 'playwright/.auth/user.json' } },
  { name: 'chromium', testIgnore: /.*\.api\.spec\.ts/, dependencies: ['setup'],
    use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/user.json' } },
],
```

The agents never edit `playwright.config.ts`. If there is no API project, they run API files with
`<smoke-projects>`' first project (the files then run once per matching project), and the planner adds
"add an `api` project to `playwright.config.ts`" to its open questions.

## Roles

`apiAs(role)` builds a request context from `<roles>`. A test that needs a role the project has no state file for
can't be written: the planner lists it as uncovered (a follow-up for the project: have `<setup-project>` save that
role's state too). Never log in from inside a test to make up for it.

## Template: an "API agents" section for a project's CLAUDE.md

```markdown
## API agents

- API base: https://app.example.com, paths under `/api/v1`. No OpenAPI file; routes live in `server/routes/*.ts`.
- Auth: cookie session from `playwright/.auth/user.json`. Roles: `admin` → `playwright/.auth/admin.json`,
  `viewer` → `playwright/.auth/viewer.json` (all written by the `setup` project).
- Error shape: `{ "error": { "code": string, "message": string, "fields"?: { [field]: string } } }`.
- Project: `--project=api` runs `tests/api/**/*.api.spec.ts`.
- Data safety: staging. Tests may create records named `qa-*` and must delete them. Never call `/api/v1/billing/*`.
- Pitfalls: `DELETE /projects/:id` is a soft delete (returns 204, `GET` then returns 410). Lists are cached for 5s.
```
