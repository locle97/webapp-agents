---
name: api-testing
description: Design, write and debug REST API tests with Playwright's request fixture. Covers finding endpoints (contract, code, network capture), probing them live through playwright-cli, the reusable test layers (clients, shapes, checks, factories, case tables), the per-endpoint check catalog and API effort levels. Preloaded by the api-test-planner, api-test-generator and api-test-healer agents.
allowed-tools: Bash(playwright-cli:*) Bash(npx:*) Bash(npm:*) Bash(curl:*)
---

# REST API testing with Playwright

API tests run on the project's existing Playwright setup: `@playwright/test`, its config, reporters and CI job. They
use the `request` fixture (`APIRequestContext`) instead of a page, and they log in through the same
`<storage-state>` the e2e tests use. There is no second runner and no new dependency.

```
tests/api/                         <api-dir>
  fixtures.ts                      <api-fixtures>: `api`, `apiAs(role)`, `cleanup`; extends the e2e fixtures
  clients/<resource>.ts            one endpoint object per resource: typed calls, no assertions
  shapes/<resource>.ts             response shapes (asymmetric matchers), plus shapes/error.ts
  checks.ts                        shared assertions: expectStatus, expectShape, expectError, expectAuthMatrix
  factories/<resource>.ts          valid payloads and create-with-cleanup helpers
  <resource>/<scenario>.api.spec.ts
```

A test only says **what** is checked. How to call an endpoint (client), what a valid record looks like (factory),
what a good response looks like (shape) and what an error looks like (checks) each live in exactly one place, so a
change to the API is fixed once, not in every test.

## Where things are

| Topic | Reference |
|---|---|
| Placeholders (`<api-base-url>`, `<api-dir>`, `<api-project>`, roles, auth) and the project `CLAUDE.md` template | [references/api-conventions.md](references/api-conventions.md) |
| The layers with code templates, the check catalog, plan format, test file rules, API data in e2e tests | [references/test-design.md](references/test-design.md) |
| Finding endpoints, the API map (`docs/apimap/`), probing requests live | [references/probing.md](references/probing.md) |
| Effort levels for API plans, generation and healing | [references/effort-levels.md](references/effort-levels.md) |

Browser commands (`open`, `state-load`, `run-code`, `requests`) come from the `playwright-cli` skill.

## Rules every API agent follows

- **Read-only by default.** Unless the mission allows data changes, send only `GET`/`HEAD`/`OPTIONS`, whether
  probing or testing. A validation probe is a write too: if validation is broken, it creates the record.
- **Everything a test creates, it deletes.** Create through a factory, which registers the delete with `cleanup`.
  Every name the tests create starts with `qa-<runId>-`, so leftovers can be found.
- **Never run the auth setup.** Read `<storage-state>` (and any role state files), never write them. Every
  `npx playwright test` command includes `--no-deps`. A 401 where the plan expects success means
  `auth: storage state missing or expired`: stop and report it.
- **Assert what the contract or spec promises.** Status, shape and the fields a requirement names. Don't pin
  timestamps, generated ids, ordering or fields nobody promised.
- **Never write secrets.** Don't write tokens, cookie values or real personal data into tests, plans, the API map or
  reports.
- **Probing sessions run headed.** `playwright-cli open --headed`, as in the `playwright-cli` skill. API test runs
  open no browser, so `--headed` makes no difference there.
