# Site Scrub

[![GitHub Marketplace](https://img.shields.io/badge/GitHub-Marketplace-blue?logo=github)](https://github.com/marketplace/actions/site-scrub)
[![License: CC0-1.0](https://img.shields.io/badge/License-CC0_1.0-lightgrey.svg)](LICENSE)

AI-powered SEO audit for static sites — scans your built HTML and delivers actionable feedback via Claude, GPT, or Gemini.

- **On pull requests** — reviews only the changed pages, posts a sticky PR comment
- **On schedule / manual run** — full-site audit with broken-link checks and trend arrows (▲/▼/▬), filed as a labelled issue

Checks titles, descriptions, canonicals, H1s, `robots` directives, Open Graph & Twitter tags, JSON-LD, image alt text, and internal links.

## Quick start

```yaml
- name: Site Scrub
  uses: datum-cloud/site-scrub@v1
  with:
    api-key: ${{ secrets.ANTHROPIC_API_KEY }}
    site-url: https://www.example.com
```

Add this step **after** your build step. That's it.

## Full workflow example

```yaml
name: Site Scrub

on:
  pull_request:
  schedule:
    - cron: '0 6 * * 0' # weekly full-site audit
  workflow_dispatch:
    inputs:
      mode:
        description: 'Scan mode'
        type: choice
        default: 'full'
        options: [changed-only, full]

concurrency:
  group: site-scrub-${{ github.ref }}-${{ github.event_name }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  site-scrub:
    if: github.event_name != 'pull_request' || github.event.pull_request.user.type != 'Bot'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: 'npm'

      - run: npm ci

      - name: Build site
        env:
          NODE_ENV: production
        run: npm run build

      - name: Site Scrub
        uses: datum-cloud/site-scrub@v1
        with:
          api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          site-url: https://www.example.com
          dist-dir: dist
          scan-mode: ${{ github.event_name == 'workflow_dispatch' && inputs.mode || 'auto' }}
```

## AI providers

Defaults to Anthropic (Claude). Switch with `provider` + `api-key`:

```yaml
# OpenAI
with:
  provider: openai
  api-key: ${{ secrets.OPENAI_API_KEY }}

# Google Gemini
with:
  provider: gemini
  api-key: ${{ secrets.GEMINI_API_KEY }}

# OpenAI-compatible gateway (OpenRouter, Groq, …)
with:
  provider: openai
  api-key: ${{ secrets.OPENROUTER_API_KEY }}
  base-url: https://openrouter.ai/api
  model: anthropic/claude-sonnet-4.6
```

Default models: `claude-sonnet-4-6` · `gpt-4o-mini` · `gemini-2.5-flash`. Override with `model`.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `api-key` | ✅ | — | API key for the selected provider. |
| `site-url` | ✅ | — | Production origin (e.g. `https://www.example.com`). Used to recognise internal links. |
| `provider` | | `anthropic` | AI provider: `anthropic`, `openai`, or `gemini`. |
| `dist-dir` | | `dist` | Directory containing built HTML. Astro adapter builds: use `dist/client`. |
| `scan-mode` | | `auto` | `auto` (changed-only on PRs, full otherwise), `changed-only`, or `full`. |
| `pages-dir` | | `src/pages` | Source dir whose files map to routes; used for changed-file detection. |
| `content-dir` | | `src/content` | Content-collection dir mapped to routes; used for changed-file detection. |
| `config-file` | | — | Path to a JSON config with `excludePaths`, `excludeFiles`, `excludeFilePatterns`. |
| `max-pages` | | `0` (unlimited) | Cap on pages analyzed per run. |
| `model` | | pinned per provider | Model ID override. |
| `base-url` | | — | API base URL override for OpenAI-compatible gateways. |
| `github-token` | | `github.token` | Token for posting PR comments and opening audit issues. |
| `broad-change-pattern` | | Astro-oriented | Regex; matching changed files skip the PR scan (layout/config edits affect all pages). |
| `anthropic-api-key` | | — | Deprecated alias of `api-key` (Anthropic only). |

## Scan modes

- **`changed-only`** (PR default) — reviews only pages whose source files changed. If no changed files are provided, the scan is skipped silently with no PR comment. PRs that touch shared layouts/config, or have no matching page changes, also skip the full-site review, but post a PR comment explaining why, since a per-page diff would be misleading or there's nothing to review.
- **`full`** (schedule/dispatch default) — reviews every built page plus broken internal links and redirect chains. Results are filed as an issue labelled `site-scrub`; the previous issue is closed and its score block is used to compute trend arrows.
- **`auto`** — selects `changed-only` on `pull_request`, `full` on everything else.

## Exclusion config

Create a JSON file and pass its path via `config-file`:

```json
{
  "excludePaths": ["/tags", "/authors"],
  "excludeFiles": ["404.html"],
  "excludeFilePatterns": ["^draft-"]
}
```

```yaml
- uses: datum-cloud/site-scrub@v1
  with:
    api-key: ${{ secrets.ANTHROPIC_API_KEY }}
    site-url: https://www.example.com
    config-file: site-scrub.config.json
```

## Requirements

- The job must checkout the repo and **build the site** before calling this action (audits files on disk, not a live URL).
- Node.js on the runner (`actions/setup-node` or the runner default).
- An API key for one of the supported providers.
- Permissions: `contents: read`, `pull-requests: write`, `issues: write`.
