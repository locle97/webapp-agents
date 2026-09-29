# Finding and probing endpoints

The e2e generator replays each step in a live browser before it writes a test. The API agents do the same with
requests: they send each request for real, read the status, headers and body, and only then write the assertion.
Nothing in a plan or test should be a guess about what the API returns.

## 1. Find the endpoints

Use these sources in order, and record where each endpoint came from:

1. **The contract.** In an orchestrated mission, the Contract's **API surface** in `intent.md` is the source of
   truth: test what it says. Probing checks that the build matches; a mismatch is a finding, not something to
   adopt.
2. **The API map.** `docs/apimap/README.md` and `docs/apimap/<area>.md`, written by earlier missions.
3. **An OpenAPI file** (`<openapi>`), if the repo has one.
4. **The server code.** Search for route definitions, for example:

   | Stack | Where routes live |
   |---|---|
   | Express / Fastify / Koa | `router.get(`, `app.post(`, `fastify.route(` |
   | Next.js | `app/**/route.ts` (exported `GET`, `POST`, ...), `pages/api/**` |
   | NestJS | `@Controller(`, `@Get(`, `@Post(` |
   | Rails | `config/routes.rb` (`resources`, `get`, `post`) |
   | Django / DRF | `urls.py`, `ViewSet`s, `@api_view` |
   | FastAPI / Flask | `@app.get(`, `@router.post(`, `@bp.route(` |
   | Spring | `@RestController`, `@GetMapping`, `@PostMapping` |
   | Laravel | `routes/api.php` |

   Read the handler for the request fields, the validation, the status codes, and the roles it checks.
5. **The frontend's own calls.** Open the feature in a browser session, use it as a user would (read-only unless
   writes are allowed), then list what it called:

   ```bash
   playwright-cli -s=api-map open --headed
   playwright-cli -s=api-map state-load <storage-state>
   playwright-cli -s=api-map goto <base-url>/projects
   playwright-cli -s=api-map requests          # every request the page made
   playwright-cli -s=api-map request 7         # headers, payload and response of one of them
   ```

   This gives real payloads and responses, and shows which endpoints the UI actually depends on.

## 2. Probe

Probe through a `playwright-cli` session that has loaded `<storage-state>`. `page.request` shares the browser
context's cookies, so it sends exactly what the tests' `default` role sends. Use absolute URLs: the session has no
`baseURL`.

```bash
playwright-cli -s=api-A1 open --headed
playwright-cli -s=api-A1 state-load <storage-state>
playwright-cli -s=api-A1 --raw run-code "async page => {
  const res = await page.request.get('<api-base-url>/api/projects?page=1');
  return { status: res.status(), type: res.headers()['content-type'], body: (await res.text()).slice(0, 4000) };
}"
```

- **Anonymous** (`unauth`): send no credentials with curl: `curl -sS -i '<api-base-url>/api/projects'`.
- **Other roles**: `state-load` that role's state file in a separate session (`-s=api-A1-viewer`).
- **Bearer auth** (`<api-auth>` = `bearer:localStorage:<key>`): `goto <base-url>` first, read the token inside
  `run-code` with `await page.evaluate(k => localStorage.getItem(k), '<key>')`, and pass it in `headers`. Never
  return or print the token.
- **Writes** (`POST`/`PUT`/`PATCH`/`DELETE`, validation probes included) only when the mission allows data changes.
  Use a `qa-<runId>-` name, and delete what you created in the same `run-code` call:

  ```bash
  playwright-cli -s=api-A1 --raw run-code "async page => {
    const created = await page.request.post('<api-base-url>/api/projects', { data: { name: 'qa-probe-1' } });
    const body = await created.json();
    const invalid = await page.request.post('<api-base-url>/api/projects', { data: {} });
    const out = { create: [created.status(), body], invalid: [invalid.status(), await invalid.json()] };
    if (created.ok()) out.deleted = (await page.request.delete('<api-base-url>/api/projects/' + body.id)).status();
    if (invalid.ok()) out.invalidDeleted = (await page.request.delete('<api-base-url>/api/projects/' + (await invalid.json()).id)).status();
    return out;
  }"
  ```

  If a delete fails, report the leftover record's id and name as an open question.
- Close the session when you are done: `playwright-cli -s=api-A1 close`, and check `playwright-cli list`.

What to record from a probe: the status, the fields and their types (not their values, unless the requirement fixes
them), the error body's format, required headers, and anything surprising (a 200 where 201 was expected, a soft
delete, caching, eventual consistency).

## 3. The API map: `docs/apimap/`

The API map is to API tests what `docs/sitemap/` is to e2e tests: what later missions read instead of rediscovering.
The API planner writes and updates it; the other agents only read it.

`docs/apimap/README.md`:

```markdown
# API map

Base: `<api-base-url>` · Auth: cookie session (`playwright/.auth/user.json`) · Roles: default, anon
Error shape: `{ error: { code, message, fields? } }`

| Area | Doc | Endpoints | Last probed |
|---|---|---|---|
| projects | [projects.md](projects.md) | 5 | 2026-09-29 |
```

`docs/apimap/<area>.md`:

```markdown
# projects

Last probed: 2026-09-29 · Code: `server/routes/projects.ts` · Client: `tests/api/clients/projects.ts`

| Method | Path | Auth | Request | Success | Errors | Source |
|---|---|---|---|---|---|---|
| GET | /api/projects | default | `?page&q` | 200 `{ items: Project[], total }` | 401 | code, probed |
| POST | /api/projects | default | `{ name*, description? }` | 201 Project | 401, 422 `fields.name` | contract v1, probed |

## Shapes
- Project: `id` string, `name` string (1-255), `description` string|null, `createdAt` ISO date

## Rules and pitfalls
- `name` is trimmed; blank after trimming → 422.
- `DELETE` is a soft delete: 204, then `GET` → 410.
```

Mark each endpoint's **Source** (`contract`, `openapi`, `code`, `observed`, plus `probed` once you have sent a
request to it). Endpoints you found but could not probe (writes on a read-only mission) stay `code` only. Never put
tokens, cookie values or real user data in the map.
