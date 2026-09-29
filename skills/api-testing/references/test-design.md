# Reusable API test design

API tests are built from five layers. Each change to the API is fixed in one place: a renamed path in the client, a
new required field in the factory, a new response field in the shape, a new error format in `shapes/error.ts`.
Tests stay short and read like the plan.

| Layer | File | Knows | Never does |
|---|---|---|---|
| Fixtures | `<api-fixtures>` | roles, auth, base URL, cleanup order | assert |
| Clients | `clients/<resource>.ts` | method, path, params, body of each endpoint | assert, or pick test data |
| Shapes | `shapes/<resource>.ts`, `shapes/error.ts` | what a good response and an error look like | call the API |
| Checks | `checks.ts` | how to assert status, shape, error, role matrix | know any one resource |
| Factories | `factories/<resource>.ts` | a valid payload; create a record and register its delete | know what a test checks |

**Add to a layer only what a scenario needs.** The first API test in a project creates `fixtures.ts`, `checks.ts`
and `shapes/error.ts`. The first test of a resource creates its client, shape and factory with the endpoints and
fields that test uses. Later tests extend them. Never copy a helper into a test file, and never write a second
client for the same resource.

The templates below use a `projects` resource and the error shape
`{ error: { code, message, fields? } }`. Replace them with the project's.

## Fixtures: `<api-fixtures>`

```ts
import type { APIRequestContext } from '@playwright/test';
// Extend the project's e2e fixtures (<fixtures>) when it has them, so e2e tests can seed data through the API.
// Otherwise import from '@playwright/test'.
import { test as base, expect } from '../fixtures';
import { ProjectsClient } from './clients/projects';

// <roles>: role -> storage state file written by the setup project. `anon` sends no credentials.
const ROLES: Record<string, string | null> = {
  default: 'playwright/.auth/user.json',
  anon: null,
};

export type Api = {
  request: APIRequestContext;
  projects: ProjectsClient;
};

export class Cleanup {
  private tasks: Array<() => Promise<unknown>> = [];

  add(task: () => Promise<unknown>) {
    this.tasks.push(task);
  }

  async run() {
    const errors: string[] = [];
    for (const task of this.tasks.reverse()) {
      try {
        await task();
      } catch (e) {
        errors.push(String(e));
      }
    }
    if (errors.length) throw new Error(`cleanup failed, qa-* data may be left behind: ${errors.join('; ')}`);
  }
}

type ApiFixtures = {
  apiAs: (role: string) => Promise<Api>;
  api: Api;
  cleanup: Cleanup;
};

export const test = base.extend<ApiFixtures>({
  apiAs: async ({ playwright, baseURL }, use) => {
    const contexts: APIRequestContext[] = [];
    await use(async role => {
      if (!(role in ROLES)) throw new Error(`no storage state for role "${role}"`);
      const request = await playwright.request.newContext({
        baseURL: process.env.API_BASE_URL ?? baseURL,
        storageState: ROLES[role] ?? undefined,
      });
      contexts.push(request);
      return { request, projects: new ProjectsClient(request) };
    });
    for (const request of contexts) await request.dispose();
  },
  api: async ({ apiAs }, use) => {
    await use(await apiAs('default'));
  },
  // Depends on apiAs so the request contexts are still open while the deletes run.
  cleanup: async ({ apiAs }, use) => {
    const cleanup = new Cleanup();
    await use(cleanup);
    await cleanup.run();
  },
});

export { expect };
```

For `<api-auth>` = `bearer:localStorage:<key>`, read the token from the role's state file and pass it as
`extraHTTPHeaders: { Authorization: \`Bearer ${token}\` }` in `newContext` (state file:
`origins[].localStorage[]` entries with `name` / `value`). Never print the token.

`baseURL` and paths: Playwright resolves paths like `new URL(path, baseURL)`, so `/api/projects` drops any path
on the base URL. Keep the base URL an origin and put the full path (`/api/v1/projects`) in the client.

## Clients: `clients/<resource>.ts`

```ts
import type { APIRequestContext } from '@playwright/test';

export type ProjectInput = { name: string; description?: string };
export type Project = ProjectInput & { id: string; createdAt: string };

export class ProjectsClient {
  constructor(private readonly request: APIRequestContext) {}

  list(params: Record<string, string | number | boolean> = {}) {
    return this.request.get('/api/projects', { params });
  }

  get(id: string) {
    return this.request.get(`/api/projects/${id}`);
  }

  // Loose input so negative tests can send missing fields and wrong types through the same client.
  create(data: Record<string, unknown>) {
    return this.request.post('/api/projects', { data });
  }

  update(id: string, data: Record<string, unknown>) {
    return this.request.patch(`/api/projects/${id}`, { data });
  }

  remove(id: string) {
    return this.request.delete(`/api/projects/${id}`);
  }
}
```

Clients return the `APIResponse` and never assert, so a happy-path test and a 403 test use the same call.

## Shapes: `shapes/<resource>.ts` and `shapes/error.ts`

Shapes are plain objects of asymmetric matchers for `toMatchObject`, so they need no schema library. Use zod or ajv
only if the project already depends on one. List the fields the contract or the frontend relies on, not every field.

```ts
// shapes/projects.ts
import { expect } from '@playwright/test';

export const ISO_DATE = /^\d{4}-\d{2}-\d{2}T/;

export const projectShape = {
  id: expect.any(String),
  name: expect.any(String),
  createdAt: expect.stringMatching(ISO_DATE),
};

export const projectPageShape = {
  items: expect.any(Array),
  total: expect.any(Number),
};
```

```ts
// shapes/error.ts: the one error format (<error-shape>)
import { expect } from '@playwright/test';

export const errorShape = (field?: string) => ({
  error: {
    code: expect.any(String),
    message: expect.any(String),
    ...(field ? { fields: { [field]: expect.any(String) } } : {}),
  },
});
```

## Checks: `checks.ts`

```ts
import { expect, type APIResponse } from '@playwright/test';
import type { Api } from './fixtures';
import { errorShape } from './shapes/error';

// Every check puts the URL and body in the failure message, so a red test explains itself.
export async function expectStatus(res: APIResponse, status: number) {
  expect(res.status(), `${res.url()} -> ${res.status()}\n${await res.text()}`).toBe(status);
}

export async function expectShape<T>(res: APIResponse, status: number, shape: Record<string, unknown>): Promise<T> {
  await expectStatus(res, status);
  const body = await res.json();
  expect(body).toMatchObject(shape);
  return body as T;
}

export function expectEach(items: unknown[], shape: Record<string, unknown>) {
  for (const item of items) expect(item).toMatchObject(shape);
}

export async function expectError(res: APIResponse, status: number, field?: string) {
  await expectShape(res, status, errorShape(field));
}

// One soft assertion per role, so a single run shows the whole matrix.
export async function expectAuthMatrix(
  apiAs: (role: string) => Promise<Api>,
  call: (api: Api) => Promise<APIResponse>,
  matrix: Record<string, number>,
) {
  for (const [role, status] of Object.entries(matrix)) {
    const res = await call(await apiAs(role));
    expect.soft(res.status(), `${role}: ${res.url()}\n${await res.text()}`).toBe(status);
  }
}
```

## Factories: `factories/<resource>.ts`

```ts
import type { APIResponse } from '@playwright/test';
import { expectShape } from '../checks';
import type { Project } from '../clients/projects';
import type { Api, Cleanup } from '../fixtures';
import { projectShape } from '../shapes/projects';

export const runId = process.env.QA_RUN_ID ?? Date.now().toString(36);
export const qaName = (label: string) => `qa-${runId}-${label}-${Math.random().toString(36).slice(2, 7)}`;

// A valid payload. Overriding a field with `undefined` drops it from the JSON body ("missing field").
export function projectInput(overrides: Record<string, unknown> = {}) {
  return { name: qaName('project'), ...overrides };
}

// Sends the create and registers a delete if the API accepted it, even when the test expected a rejection.
export async function attemptCreateProject(api: Api, cleanup: Cleanup, input: Record<string, unknown>): Promise<APIResponse> {
  const res = await api.projects.create(input);
  if (res.ok()) {
    const { id } = await res.json();
    cleanup.add(() => api.projects.remove(id));
  }
  return res;
}

// For preconditions: create a valid record or fail.
export async function createProject(api: Api, cleanup: Cleanup, overrides: Record<string, unknown> = {}) {
  const res = await attemptCreateProject(api, cleanup, projectInput(overrides));
  return expectShape<Project>(res, 201, projectShape);
}
```

Write tests always go through `attempt*` or `create*`, never straight through the client, so nothing leaks.

## The check catalog

The planner builds scenarios from this list instead of inventing them. Each scenario names its check with a
`**Check:**` line. [effort-levels.md](effort-levels.md) says which checks each level includes. A check a requirement
names explicitly is in scope at any level.

| Check | Proves | Typical expectation |
|---|---|---|
| `happy` | A valid request works | success status, body matches the shape, the fields the requirement names have the sent or documented values |
| `unauth` | The endpoint needs credentials | `anon` → 401 (or what the contract says) |
| `validation` | Bad input is rejected with a usable error | one case per changed input: missing, wrong type, invalid value → 4xx + error shape naming the field |
| `persist` | A write is stored | the following `GET` (and the list, if there is one) returns the new state |
| `not-found` | Unknown ids are handled | unknown or deleted id → 404 (or 410) + error shape |
| `authz` | Each role gets what it is allowed | role matrix: every role in `<roles>` → its status |
| `boundary` | Limits hold | min/max length, empty, whitespace, unicode, numeric limits: accepted or rejected as documented |
| `conflict` | Uniqueness and concurrent changes | duplicate unique value → 409; stale update → 409/412 if the API versions records |
| `list` | Pagination, filters and sort work | page size honored, `total` right, filter narrows, sort order holds (on `qa-` data the test created) |
| `idempotency` | Repeats are safe | repeating a `PUT`/`DELETE` gives the documented result |
| `side-effect` | A write changes what else it should | counts, related resources or other endpoints reflect it |

Several cases of the same check against the same endpoint (the validation of each field, the rows of a boundary
check) are **one scenario with a case table**, not one scenario per case.

## Plan format: `specs/<feature>.api.plan.md`

```markdown
# Projects API test plan

**Effort:** medium
**API map:** `docs/apimap/projects.md`
**Source:** contract v1 (`docs/missions/2026-09-29-projects/intent.md`)

## Endpoints in scope
| Method | Path | Auth | Success | Errors |
|---|---|---|---|---|
| POST | /api/projects | default | 201 Project | 401, 422 |
| GET | /api/projects/{id} | default | 200 Project | 401, 404 |

### A1. Projects: create
**Endpoint:** `POST /api/projects`

#### A1.1 create-project-returns-the-new-project
**File:** `tests/api/projects/create-project-returns-the-new-project.api.spec.ts`
**Covers:** AC-2
**Check:** happy, persist
**Data:** creates 1 project (factory, deleted in cleanup)
**Steps:**
1. As `default`, POST /api/projects with a valid payload
   - expect: 201, body matches the Project shape
   - expect: `name` equals the sent name
2. GET /api/projects/{id} of the new project
   - expect: 200 with the same `name`

#### A1.2 create-project-rejects-invalid-input
**File:** `tests/api/projects/create-project-rejects-invalid-input.api.spec.ts`
**Covers:** AC-3
**Check:** validation
**Data:** writes only if validation is broken (attempt, deleted in cleanup)
**Steps:**
1. As `default`, POST /api/projects with each input below
   - expect: the status, and the error shape naming the field
**Cases:**
| Case | Input | Status | Field |
|---|---|---|---|
| missing name | `name` omitted | 422 | name |
| blank name | `name: "   "` | 422 | name |

## Deferred scenarios
- `create-project-role-matrix`: authz, needs a `viewer` state file the setup project doesn't write
```

Scenario ids start with `A` so they never collide with e2e ids in the same mission. Keep ids and file paths stable
when a plan is extended.

## Test files

- One scenario per file: `<api-dir>/<resource>/<scenario-name>.api.spec.ts`, from the `**File:**` line.
- Headers: `// spec: <plan path>` and `// endpoint: <METHOD path>`.
- `test.describe` named after the group without its ordinal. The test title is the scenario name; in a case table
  each row is `<scenario name>: <case>`.
- A comment with the step text before each step, and `// - expect:` comments before its assertions.
- Import `test` and `expect` from `<api-fixtures>`, never from `@playwright/test`.
- No sleeps. When an API is eventually consistent, use `expect.poll(() => ...)` with the timeout the contract or API
  map documents.

```ts
// spec: specs/projects.api.plan.md
// endpoint: POST /api/projects
import { test, expect } from '../fixtures';
import { expectShape } from '../checks';
import { attemptCreateProject, projectInput } from '../factories/projects';
import type { Project } from '../clients/projects';
import { projectShape } from '../shapes/projects';

test.describe('Projects: create', () => {
  test('create-project-returns-the-new-project', async ({ api, cleanup }) => {
    // 1. As `default`, POST /api/projects with a valid payload
    const input = projectInput();
    const res = await attemptCreateProject(api, cleanup, input);
    // - expect: 201, body matches the Project shape
    const project = await expectShape<Project>(res, 201, projectShape);
    // - expect: `name` equals the sent name
    expect(project.name).toBe(input.name);

    // 2. GET /api/projects/{id} of the new project
    // - expect: 200 with the same `name`
    const fetched = await expectShape<Project>(await api.projects.get(project.id), 200, projectShape);
    expect(fetched.name).toBe(input.name);
  });
});
```

```ts
// spec: specs/projects.api.plan.md
// endpoint: POST /api/projects
import { test } from '../fixtures';
import { expectError } from '../checks';
import { attemptCreateProject, projectInput } from '../factories/projects';

const cases = [
  { name: 'missing name', input: { name: undefined }, status: 422, field: 'name' },
  { name: 'blank name', input: { name: '   ' }, status: 422, field: 'name' },
];

test.describe('Projects: create', () => {
  for (const c of cases) {
    test(`create-project-rejects-invalid-input: ${c.name}`, async ({ api, cleanup }) => {
      // 1. As `default`, POST /api/projects with each input below
      const res = await attemptCreateProject(api, cleanup, projectInput(c.input));
      // - expect: the status, and the error shape naming the field
      await expectError(res, c.status, c.field);
    });
  }
});
```

A role matrix on a read endpoint:

```ts
await expectAuthMatrix(apiAs, a => a.projects.list(), { default: 200, anon: 401 });
```

On a write endpoint, send each role through `attempt*` so the roles that succeed clean up after themselves.

## API data in e2e tests

When the mission allows creating data, e2e tests set up their preconditions through the API instead of through the
UI: it's faster, it doesn't depend on other screens working, and the factories clean up. Because
`<api-fixtures>` extends `<fixtures>`, one import gives both `page` and `api`:

```ts
// spec: specs/projects.plan.md
// seed: tests/seeds/projects.seed.spec.ts
import { test, expect } from '../api/fixtures';
import { createProject } from '../api/factories/projects';

test.describe('Project list', () => {
  test('new-project-shows-in-the-list', async ({ page, api, cleanup }) => {
    // Given a project exists (API)
    const project = await createProject(api, cleanup);
    // 1. Open the projects page
    await page.goto('/projects');
    // - expect: the project is listed
    await expect(page.getByRole('link', { name: project.name })).toBeVisible();
  });
});
```

`api` logs in with the same `<storage-state>` as `page`, so both see the same user's data. Only factories that
already exist are used this way: an e2e agent that needs a missing one reports it instead of writing it.
