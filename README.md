# Faultline AI Trust & Safety Scanner

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Faultline-blue?logo=github)](https://github.com/marketplace/actions/faultline-ai-trust-safety-scanner)
[![License](https://img.shields.io/badge/License-Apache_2.0-green.svg)](LICENSE)

Forensic AI output verification for your CI pipeline. Detects hallucination, manipulation, and policy violations in AI-generated content — before it ships.

Results appear as **inline PR annotations** and **Security tab alerts** via GitHub Code Scanning.

---

## Usage

```yaml
# .github/workflows/faultline.yml
name: Faultline AI Safety Scan

on:
  push:
    branches: [main]
  pull_request:

permissions:
  security-events: write
  contents: read
  actions: read

jobs:
  faultline:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Faultline AI Trust & Safety Scan
        uses: nxtg-ai/faultline-action@v1
        with:
          input: 'docs/ai-output.md'
          fail-on: 'high'
        env:
          GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
```

SARIF results are automatically uploaded to GitHub Code Scanning — no extra steps needed.

---

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `mode` | `scan` (a file in the repo) or `guard` (a block of text) | `scan` |
| `input` | scan mode — file to scan (repo-relative path) | `README.md` |
| `text` | guard mode — text to check. Treated as data, never as shell input. | _(empty)_ |
| `fail-on` | scan mode — minimum severity to fail the build: `critical`, `high`, `medium`, `low` | `high` |
| `guard-fail-on` | guard mode — `refuted` or `unsupported`. Empty means advisory. | _(empty)_ |
| `provider` | AI provider: `gemini`, `openai`, `claude`, `perplexity`, `mock`. Auto-detected from env vars if not set. | _(auto)_ |
| `version` | Version of `@nxtg/faultline` to install. Pin for reproducible runs. | `latest` |
| `output-path` | scan mode — path to write SARIF output | `faultline.sarif` |
| `upload-sarif` | Upload SARIF to code scanning. Needs `security-events: write`. | `true` |

---

## Outputs

| Output | Description |
|--------|-------------|
| `sarif-file` | scan mode — path to the generated SARIF file |
| `verdict-count` | scan mode — number of results in the report |
| `highest-severity` | scan mode — highest severity found: `high`, `medium`, `low`, `none` |
| `guard-passed` | guard mode — `"true"` when the gate passed |
| `guard-json` | guard mode — path to the JSON verdict report |

---

## Guard mode — check a PR body

`scan` mode checks a file. `guard` mode checks a block of text, which is what you
want for a PR description, release notes, or an agent's summary of its own work.

```yaml
name: Check PR description

on:
  pull_request:
    types: [opened, edited, synchronize]

permissions:
  contents: read

jobs:
  check-pr-body:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: nxtg-ai/faultline-action@v1
        with:
          mode: 'guard'
          text: ${{ github.event.pull_request.body }}
          guard-fail-on: 'refuted'      # omit to report without failing
        env:
          GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
```

Without `guard-fail-on` the step reports and always passes. With it, the step
fails on a refuted claim — **and also fails when verification was degraded**, on
the grounds that claims which were never checked are not claims that passed.

The PR body is passed to the action as data. It is written to a file with
`printf %s` and piped in; it is never interpolated into a shell command, so a
body containing `$(...)` or backticks cannot execute.

---

## How Results Appear

After the action runs, violations appear in two places:

1. **Security → Code scanning alerts**: Every finding with file and line context
2. **PR → Files changed**: Inline annotations on the exact lines with issues

Findings are categorised by the Faultline rule that fired:
- `faultline/verification/contradicted` — claim contradicted by evidence
- `faultline/verification/mixed` — claim has mixed supporting evidence
- `faultline/eu-ai-act/*` — EU AI Act risk tier detections (Article 5, Annex III, Article 50)
- Custom rules: `pii`, `bias`, `toxicity`, `manipulation`, and more

---

## Modes

### Rules only (`provider: mock` — no API key required)

```yaml
- uses: nxtg-ai/faultline-action@v1
  with:
    input: 'README.md'
    provider: 'mock'
    fail-on: 'critical'
```

Runs the deterministic rule checks — PII, toxicity, bias, shell injection, EU AI
Act risk tiers — with no external API call. Zero cost, instant, and enough to
enforce policy on generated content.

**It does not verify any claim.** The `mock` provider returns `supported` for
every claim it is given. That is fine for rule enforcement and for wiring up CI,
but a mock run can never disagree with the text, so do not read a green mock run
as "the claims check out."

### Live AI Verification (API key required)

Set your provider's API key as a repository secret, then pass it as an env var:

```yaml
- uses: nxtg-ai/faultline-action@v1
  with:
    input: 'docs/ai-output.md'
    fail-on: 'high'
  env:
    GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}     # Gemini (recommended — free tier available)
    OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}     # GPT-4o-mini
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }} # Claude
    PERPLEXITY_API_KEY: ${{ secrets.PERPLEXITY_API_KEY }} # Perplexity Sonar (with citations)
```

The provider is auto-detected from whichever key is present. Use `provider: 'gemini'` to force a specific provider.

**Use Gemini if you want evidence.** It is the only provider that retrieves live
sources while verifying (via the `googleSearch` tool). The others judge from the
model's own knowledge and return verdicts with no source URLs attached — useful
as a signal, but not evidence you can show anyone.

Get a free Gemini API key at [aistudio.google.com](https://aistudio.google.com/apikey).

---

## What this repo's own CI proves

The workflow in `.github/workflows/test.yml` runs the action against fixtures in
`fixtures/`, and is deliberately split by what each job can actually establish:

| Job | Proves | Needs a secret |
|---|---|---|
| `RED — seeded-bad fixture must fail` | The gate fails on bad content | No |
| `GREEN — clean fixture must pass` | Clean content passes, with a report | No |
| `SARIF — valid report` | The report is well-formed and not empty for bad content | No |
| `GUARD — text mode` | Guard runs, stays advisory, and treats input as data (injection canary) | No |
| `LIVE — real verification` | A known-false claim is actually refuted against sources | Yes — `GEMINI_API_KEY` |

The first four cover the **rules** engine and run on every PR. Only the last one
exercises **verification**, because the `mock` provider always returns
`supported` and so can never produce a red. Without the `GEMINI_API_KEY` secret
that job emits a warning saying real verification was not exercised, rather than
passing quietly and leaving the impression it was.

---

## Required Permissions

```yaml
permissions:
  security-events: write   # upload SARIF to Code Scanning
  contents: read           # checkout the repo
  actions: read            # read action metadata
```

---

## License

Apache 2.0 — see [LICENSE](LICENSE).

Built by [NXTG.ai](https://nxtg.ai).
