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
| `input` | File to scan (repo-relative path) | `README.md` |
| `fail-on` | Minimum severity to fail the build: `critical`, `high`, `medium`, `low` | `high` |
| `provider` | AI provider: `gemini`, `openai`, `claude`, `perplexity`, `mock`. Auto-detected from env vars if not set. | _(auto)_ |
| `output-path` | Path to write SARIF output | `faultline.sarif` |

---

## Outputs

| Output | Description |
|--------|-------------|
| `sarif-file` | Path to the generated SARIF file |
| `verdict-count` | Total number of verdicts returned |
| `highest-severity` | Highest severity found: `high`, `medium`, `low`, `none` |

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

### Free (mock/local — no API key required)

```yaml
- uses: nxtg-ai/faultline-action@v1
  with:
    input: 'README.md'
    provider: 'mock'
    fail-on: 'critical'
```

Runs EU AI Act rule checks and structural analysis without calling any external AI API. Zero cost, instant results. Good for enforcing EU AI Act policy and PII/manipulation rules.

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

Get a free Gemini API key at [aistudio.google.com](https://aistudio.google.com/apikey).

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
