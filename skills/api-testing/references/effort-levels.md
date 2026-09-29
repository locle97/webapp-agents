# API effort levels (low · medium · high)

Same levels, same meaning and same hard scenario caps as the e2e effort levels (the `playwright-cli` skill's
`references/effort-levels.md`): **low 5 · medium 8 · high 15**. The cap applies to each plan on its own, so a
mission with both layers can have up to 8 API and 8 e2e scenarios at `medium`. Everything that file says about
caps, `**Covers:**` lines, `## Deferred scenarios` and "scope too big" applies here too.

A scenario is one test file. A case table counts as one scenario; its rows are limited below.

| | **low** | **medium** | **high** |
|---|---|---|---|
| **Checks in scope** | `happy` (status + shape) per endpoint a requirement names | + `unauth` once per endpoint, `validation` for each input the mission changes, `persist` for each write | + `authz`, `boundary`, `not-found`, `conflict`, `list`, `idempotency`, `side-effect` where the endpoint has them |
| **Case table rows** | none | one row per changed input (missing or invalid) | as many as the boundaries need, at most 12 per table |
| **Planner discovery** | contract / API map first; probe only the endpoints in scope with `GET` | + read the handlers of the endpoints in scope; probe their errors | + the whole area's endpoints; update `docs/apimap/<area>.md` fully |
| **Generator** | assert the `expect:` bullets. Add to a layer only what this scenario needs | same | may also generalise a layer (a shared check, a factory option) when two or more tests need it |
| **Healer** | **1 attempt.** Fix paths, params, payloads and waits only | **2 attempts.** Can also fix shapes and expected values | **2 attempts.** Can also restructure the test or a layer |
| **Verification** | once | once | twice (catch order-dependent and flaky tests) |

A check a requirement names explicitly (for example "viewers can't delete") is in scope at every level.

## Example: a create-project endpoint

- **low (1):** `create-project-returns-the-new-project`.
- **medium (4):** the one above with its persist step, `create-project-requires-login`,
  `create-project-rejects-invalid-input` (2 rows), `get-project-returns-the-project`.
- **high (8):** those, plus `create-project-name-boundaries` (8 rows), `create-project-role-matrix`,
  `create-project-rejects-duplicate-name`, `get-unknown-project-returns-404`.
