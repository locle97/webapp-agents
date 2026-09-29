# Effort levels (low · medium · high)

Effort sets **how much testing** a mission does: how many scenarios, how deep, how many fix attempts. It is not the
model's reasoning effort. It exists to stop over-engineering: a one-field form does not need 14 scenarios.

The planner, generator, healer and QA manager all follow this file. `playwright-site-explorer` does not take an
effort level.

REST API tests use the same levels and caps, with API-specific scope: see the `api-testing` skill's
`references/effort-levels.md`. The cap applies to each layer's plan on its own.

## Setting the level

- The user usually picks the level when starting a mission: `/qa-pipeline [low|medium|high] <mission>`.
- The QA manager uses that level (or picks one at INTAKE when none was given), records it in the mission log header
  and under **Constraints**, and passes it to every subagent prompt as `<effort>low|medium|high</effort>`.
- **Default: `medium`.** Use `low` when the user asks for a smoke or quick check. Use `high` only when the user asks
  for thorough, regression-grade or edge-case coverage.
- A subagent whose prompt has no `<effort>` tag uses `medium`.
- The planner writes the level into the plan as a `**Effort:**` line under the title.

## Rules at every level

Effort never relaxes these:

- Data safety: no data changes without the user's permission, and tests that change data restore it in `finally`.
- File conventions (`// spec:` and `// seed:` headers, one test per file, describe named after the group), and the
  report contract.
- Never skip or delete a test to get to green. `test.fixme()` only with a recorded reason.
- Every requirement maps to at least one scenario.

## The levels

| | **low**: "does it work?" | **medium**: "does it meet the requirements?" | **high**: "does it hold up?" |
|---|---|---|---|
| **Goal** | Smoke check: the main path works | Every requirement is checked, plus the obvious ways it can fail | Regression-grade coverage of the feature |
| **Scenario cap (hard)** | **5**, one per requirement | **8** | **15** |
| **What's in scope** | Happy paths only. Negative or edge cases only when a requirement names them | Happy paths, plus **one** validation/negative case per input the mission changes, plus **one** persistence check (reload or URL) per change of state | Everything in medium, plus boundaries (empty, long, unicode, whitespace), effects on other pages, navigation and discard, error states, locale formats |
| **Planner exploration** | Sitemap first. Check only the elements the scenarios use; no full inventory | Explore the target page and the forms it directly uses | Explore the area fully and fix the sitemap doc where it's wrong |
| **Generator** | Assert only the `expect:` bullets. Use existing `<helpers-dir>` helpers; add none | Same, plus a stability assert where a step needs it (for example, wait for the URL after a save) | May add a helper to `<helpers-dir>/` when two or more tests need it |
| **Healer** | **1 attempt.** Locator and timing fixes only; anything else becomes `test.fixme()` or `needs-decision` | **2 attempts.** Can also fix assertions and expected values | **2 attempts.** Can also restructure the test to make it more reliable |
| **Manager: verification** | Chromium only | Chromium only | All projects in `playwright.config.ts`, run twice to catch flaky tests |
| **Manager: coverage follow-ups** | None. Gaps go under **Follow-ups** | One follow-up round with the planner | One follow-up round with the planner |

"Chromium only" means `<smoke-projects>` (see [project-conventions.md](project-conventions.md)): `--project=chromium`,
plus any other Chromium project that runs files the `chromium` project ignores.

## Scenario caps are hard limits

- The cap counts the scenarios the mission adds or changes. Scenarios that already exist in the spec and are left
  untouched do not count.
- Each scenario carries a `**Covers:**` line: the requirement id or the risk it covers, in one line. A scenario that
  cannot name one does not belong in the plan.
- When there are more candidate scenarios than the cap allows, keep the ones with the most value and list the rest
  under a `## Deferred scenarios` heading at the end of the plan (name and one-line reason each). Deferred scenarios
  get no `**File:**` line and are not generated.
- **Scope too big.** If the requirements cannot all be covered within the cap (for example 7 requirements at `low`),
  the planner never goes over the cap. It covers the requirements it can, lists the rest under
  `## Deferred scenarios`, and raises it in `open_questions`: how many requirements are uncovered, and the options
  (raise the effort level, split the mission, or drop requirements). The manager then asks the user before
  generating. At `high` the only options are splitting the mission or dropping requirements.

## Definition of done

**low**
- Every requirement has one happy-path scenario, and the plan has at most 5.
- Every case ends `passed`, `fixme` (with a reason) or `blocked`.
- The final report lists the categories that were not tested (negative, edge, persistence) so the user can ask for a
  higher level.

**medium**
- Everything in low, except a requirement may have more than one scenario.
- Every input the mission changes has one negative case, and every change of state has one persistence check.
- The plan has at most 8 scenarios.
- Every case passes in Chromium, or has a recorded `fixme` or decision.

**high**
- Everything in medium.
- Boundary and cross-page scenarios are in the plan, each with its `**Covers:**` reason.
- The sitemap doc for the area is up to date.
- The plan has at most 15 scenarios.
- Every case passes in every project, on both verification runs.

## Example: a profile name form (`specs/settings-profile-name.plan.md`)

- **low (2):** `update-first-and-last-name-together`, `updated-name-persists-after-reload`.
- **medium (6):** the two above, plus `update-first-name-only`, `update-last-name-only`,
  `empty-first-name-is-accepted-without-error`, `navigating-away-without-saving-discards-unsaved-changes`.
- **high (14):** all the current scenarios.
