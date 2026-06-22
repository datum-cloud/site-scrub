# Site Scrub

AI-assisted SEO & meta review for static sites, packaged as a GitHub composite action.

It scans your **built HTML output** with Cheerio (titles, descriptions, canonicals, H1s, robots, Open Graph/Twitter tags, JSON-LD, image alt text), computes a deterministic score table, and sends a compact report to an AI model (Claude, GPT, or Gemini) for an actionable review. Then it:

- **On `pull_request`** — reviews only the pages affected by the PR and posts a sticky comment.
- **On `schedule` / `workflow_dispatch`** — runs a full-site audit (including broken internal links and redirect-chain checks), opens a labelled issue, and shows trend arrows vs. the previous audit.

## Requirements

- The job must run `actions/checkout` and **build the site** before calling this action (it audits files on disk, not a live URL — SSR-only sites without prerendered HTML are not supported).
- Node.js available on the runner (`actions/setup-node` or the runner default).
- An API key secret for one of the supported AI providers (Anthropic, OpenAI, or Google Gemini).
- Permissions: `contents: read`, `pull-requests: write`, `issues: write`.

## Usage

```yaml
name: SEO Review

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
  group: seo-review-${{ github.ref }}-${{ github.event_name }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  seo-review:
    if: github.event_name != 'pull_request' || github.event.pull_request.user.type != 'Bot'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-node@v6
        with:
          node-version: 24
          cache: 'npm'

      - run: npm ci

      - name: Build site
        env:
          NODE_ENV: production
        run: npm run build

      - name: SEO review
        uses: datum-cloud/site-scrub@v1
        with:
          api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          site-url: https://www.example.com
          dist-dir: dist
          scan-mode: ${{ github.event_name == 'workflow_dispatch' && inputs.mode || 'auto' }}
```

## AI providers

The action defaults to Anthropic (Claude). Switch providers with `provider` + `api-key`:

```yaml
# OpenAI
with:
  provider: openai
  api-key: ${{ secrets.OPENAI_API_KEY }}

# Google Gemini
with:
  provider: gemini
  api-key: ${{ secrets.GEMINI_API_KEY }}

# Any OpenAI-compatible gateway (OpenRouter, Groq, …)
with:
  provider: openai
  api-key: ${{ secrets.OPENROUTER_API_KEY }}
  base-url: https://openrouter.ai/api
  model: anthropic/claude-sonnet-4.6
```

Default models per provider: `claude-sonnet-4-6` (anthropic), `gpt-5-mini` (openai), `gemini-2.5-flash` (gemini). Override with `model`.

The legacy `anthropic-api-key` input still works and implies `provider: anthropic`.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `api-key` | ✅ | — | API key for the selected provider. |
| `provider` | | `anthropic` | AI provider: `anthropic`, `openai`, or `gemini`. |
| `base-url` | | per provider | API base URL override (e.g. an OpenAI-compatible gateway with `provider: openai`). |
| `anthropic-api-key` | | — | Deprecated alias of `api-key` (Anthropic only). |
| `site-url` | ✅ | — | Production origin (e.g. `https://www.example.com`). Used to recognise internal links and render absolute links in the report. |
| `dist-dir` | | `dist` | Directory containing built HTML (Astro: `dist/client` when using an adapter). |
| `pages-dir` | | `src/pages` | Source dir whose files map to routes; used for changed-file detection and broken-link validation. |
| `content-dir` | | `src/content` | Content-collection dir mapped to routes; used for changed-file detection. |
| `scan-mode` | | `auto` | `auto` (changed-only on PRs, full otherwise), `changed-only`, or `full`. |
| `config-file` | | — | Path to a JSON config with `excludePaths`, `excludeFiles`, `excludeFilePatterns` (regex strings). |
| `broad-change-pattern` | | Astro-oriented | Regex; a changed file matching it makes the PR scan skip as a "broad change" (layout/config edits would invalidate a per-page diff). |
| `model` | | pinned in action | Model id override for the selected provider. |
| `max-pages` | | `0` (unlimited) | Cap on pages analyzed. |
| `github-token` | | `github.token` | Token for the PR comment and audit issue. |

## Exclusion config example

```json
{
  "excludePaths": ["/tags", "/authors"],
  "excludeFiles": ["404.html"],
  "excludeFilePatterns": ["^draft-"]
}
```

Pass its path via `config-file: seo-review.config.json`.

## How scan modes work

- **changed-only** (PR default): only pages whose `pages-dir`/`content-dir` sources changed are reviewed. PRs touching shared layouts/config (per `broad-change-pattern`) skip the scan with an explanatory comment, since a per-page diff would be misleading. No link audits — kept off the PR hot path.
- **full** (schedule/dispatch default): every built page is reviewed, plus broken-internal-link and redirect-chain audits. The result is filed as an issue labelled `seo-audit`; the previous issue is closed and its embedded score block is used to render trend arrows (▲/▼/▬).
