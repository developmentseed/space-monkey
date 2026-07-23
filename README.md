# 🐒🚀 space-monkey

LLM-driven exploratory ("monkey") testing for deployed web apps, as a reusable
GitHub Actions workflow. Named for the [space monkeys](https://en.wikipedia.org/wiki/Monkeys_and_apes_in_space) in the 1950s and 1960s,
who tested things.

On each run, an LLM agent (via [OpenCode](https://opencode.ai) +
[Playwright MCP](https://github.com/microsoft/playwright-mcp)) opens your deployed app in a real browser and tests it adversarially: clicking through pages, submitting invalid forms, probing forbidden routes, watching the console — guided by your app-specific context and, on pull requests, the PR diff. It writes an issue-by-issue report to the job summary and (optionally) a sticky PR comment.

The test is **non-deterministic by design**: it explores differently each run, which is the point — it finds things scripted tests don't. It should be used as part of a larger testing and review strategy, not as a replacement for deterministic tests.

By default it never blocks merges, but you can configure it to fail the job when issues are found, so branch protection can gate merges.

## Quick start

1. Create an [OpenRouter](https://openrouter.ai) API key and add it to your repo as the `OPENROUTER_API_KEY` secret. Consider setting a spend limit and (optionally) model/provider/data-retention restrictions on the key itself.
2. Add a workflow to your repo (see [examples/consumer-workflow.yml](examples/consumer-workflow.yml)):

```yaml
name: Monkey Test
on:
  workflow_dispatch:
  pull_request:
    branches: [staging] # or main, a release branch, a tag - however you cut a deploy

permissions:
  contents: read
  pull-requests: write # only needed when pr_comment: true

jobs:
  monkey-test:
    uses: developmentseed/space-monkey/.github/workflows/monkey-test.yml@v0
    with:
      base_url: ${{ vars.TEST_TARGET_URL }}
      context: tests/monkey/context.md
      pr_comment: true
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
      TEST_CREDENTIALS: ${{ secrets.TEST_CREDENTIALS }}
```

You choose when it runs: the `on:` block lives in _your_ workflow. Some choices include: `workflow_dispatch` (always keep this so you can run it manually), PRs into a release branch, or a cron schedule.

## Inputs

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `base_url` | string | _(required)_ | URL of the deployed app to test — staging, a preview deploy, production, whatever's reachable. No environment tiers are assumed. |
| `model` | string | `openrouter/auto` | OpenRouter model slug (e.g. `google/gemini-3.1-flash-lite`). The default uses [OpenRouter auto-routing](https://openrouter.ai/openrouter/auto), so restrictions configured on your API key (allowed models/providers, data retention) govern what runs. |
| `context` | string | `''` | App-specific testing context appended to the base prompt: UI quirks (e.g. "a feedback modal appears on load — dismiss it"), how to sign out, areas to focus on. Accepts inline text **or** a path to a file in your repo. |
| `pr_comment` | boolean | `false` | Post the report as a sticky PR comment (updated in place on re-runs). Requires `pull-requests: write`. |
| `timeout_minutes` | number | `45` | Job timeout. |
| `fail_on_issues` | boolean | `false` | Fail the job when the report contains `ISSUE` blocks, so branch protection can gate merges. Off by default because runs are non-deterministic. |

## Secrets

| Secret | Required | Description |
| --- | --- | --- |
| `OPENROUTER_API_KEY` | yes | OpenRouter API key. |
| `TEST_CREDENTIALS` | no | Free-form text listing test accounts, one per line, e.g. `reviewer: alice@example.com / hunter2 (can approve submissions)`. If provided, the agent signs in as each account and probes role boundaries; if omitted, it tests as an anonymous visitor. |

## Outputs

| Output | Description |
| --- | --- |
| `issue_count` | Number of `ISSUE` blocks in the report. |

## Where results go

- **Job summary** — always, on the run's page in the Actions tab.
- **Artifact** — the raw `monkey-test-report.md`, always.
- **Sticky PR comment** — when `pr_comment: true` and the run has PR context. One comment per PR, updated in place on subsequent runs.

## Writing good `context`

The base prompt is deliberately generic. The `context` input is where the test is tailored to your app. It should include:

- What the app is for, in a sentence, so the agent tests realistic journeys.
- UI quirks: modals that appear on load, how to sign out, non-obvious flows.
- Role expectations: what each test account should and should _not_ be able to do (put the credentials themselves in `TEST_CREDENTIALS`).
- Known issues to ignore or deprioritize

## Security notes

> [!CAUTION]
> Never call this workflow from `pull_request_target` — it would expose your secrets to fork PRs. Use plain `pull_request` or `workflow_dispatch`.

> [!CAUTION]
> Use throwaway test accounts against non-production deployments only. The agent is instructed to be adversarial; assume anything those accounts can do, it may do — including against real data if pointed at production.

- The LLM step runs with **no GitHub token**: only `OPENROUTER_API_KEY` is in its environment, and OpenCode's shell/file tools are disabled — the agent can only drive the browser.
- Set a spend limit on the OpenRouter key. A run is capped at
  `timeout_minutes`, and superseded runs on the same ref are auto-cancelled.

## Using from other repos and orgs

This repo is currently **private**, so only `developmentseed` repos can call it, and only after enabling access: Settings → Actions → General → Access → "Accessible from repositories in the 'developmentseed' organization".

For client-org repos, this repo must be made **public**; private workflows cannot be shared across orgs on our plan. Client orgs that restrict allowed actions must also add `developmentseed/space-monkey@*` to their allowlist, and bring their own `OPENROUTER_API_KEY` secret.

## Versioning

Semver releases are planned.
