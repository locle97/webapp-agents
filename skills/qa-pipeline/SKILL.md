---
name: qa-pipeline
description: Run a QA mission (REST API and/or e2e tests) with the playwright-qa-manager agent at a chosen effort level (low, medium or high).
argument-hint: "[low|medium|high] <goal, requirements or Jira ticket>"
disable-model-invocation: true
---

Run this QA mission through the `playwright-qa-manager` agent.

Arguments: $ARGUMENTS

1. If the first word is `low`, `medium` or `high`, that is the effort level and the rest is the mission. Otherwise
   the effort is `medium` and all of it is the mission. If there is no mission, ask the user for one and stop.
2. Spawn one `webapp-agents:playwright-qa-manager` subagent with this prompt:

   ```
   <effort>{effort}</effort>

   {mission, verbatim}
   ```

3. If it comes back BLOCKED with questions, ask the user, then send the answers to the same manager with SendMessage
   (or, if that is unavailable, spawn a new manager naming the mission id so it resumes from its mission log).
4. When it finishes, give the user its final report as it is.

Never spawn `playwright-site-explorer` from this skill, even when the report says an area has no sitemap doc or a
shallow one. The planner explores what it needs itself. Pass the follow-up on to the user, who can run the explorer
separately.
