<div align="center">
  <img src="https://raw.githubusercontent.com/OpticDiff/.github/main/assets/opticdiff-banner.svg" alt="OpticDiff code-reviewer-action" width="100%" />

  <br />
  <br />

  <h1>code-reviewer-action</h1>
  <p><strong>Reusable GitHub Action for fast, local-first AI code reviews powered by <a href="https://github.com/OpticDiff/code-reviewer">OpticDiff/code-reviewer</a>.</strong></p>

  <p>
    <a href="https://github.com/OpticDiff/code-reviewer-action/releases"><img src="https://img.shields.io/github/v/release/OpticDiff/code-reviewer-action?color=blue&label=action" alt="Action Release" /></a>
    <a href="https://github.com/OpticDiff/code-reviewer/releases"><img src="https://img.shields.io/github/v/release/OpticDiff/code-reviewer?color=blue&label=cli" alt="CLI Release" /></a>
    <a href="LICENSE"><img src="https://img.shields.io/github/license/OpticDiff/code-reviewer-action" alt="License" /></a>
    <a href="https://github.com/marketplace"><img src="https://img.shields.io/badge/marketplace-code--reviewer--action-blue?logo=github" alt="Marketplace" /></a>
  </p>
</div>

---

Analyzes pull request diffs, provides repo-aware Tree-sitter context, eliminates hallucinations via multi-model consensus, flags bugs and security vulnerabilities, and posts actionable inline comments with native GitHub suggestion blocks.

- 🔒 **Zero 3rd-Party SaaS Lock-in**: Run self-hosted with Ollama or vLLM directly on runner, or via private cloud endpoints (Vertex AI, AWS Bedrock).
- 🎯 **Multi-Model Consensus**: Run multiple models (e.g. Gemini + Claude) and only post findings they both agree on.
- ⚡ **Smart Diff Caching**: Cryptographically hashes diffs so re-runs on pushed PRs only analyze new changes.
- 🛡️ **Safe Auto-Approve**: Automatically approve clean PRs using 9 strict safety guards (SHA pinned).
- 📊 **GitHub Security Tab**: Native SARIF 2.1.0 report generation and Code Scanning upload.

---

## Quick Start (Zero Auth with Ollama)

Run a local or runner-hosted Ollama model without needing any cloud credentials or API keys:

```yaml
name: Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Start Ollama & pull model
        run: |
          curl -fsSL https://ollama.com/install.sh | sh
          ollama serve &
          sleep 5
          ollama pull qwen3:8b

      - uses: OpticDiff/code-reviewer-action@v1
        with:
          model: qwen3:8b
          extra-args: --api-url http://localhost:11434/v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Authentication & Provider Examples

### 1. Vertex AI (Recommended)

Authenticate using Google Cloud Workload Identity Federation (WIF) — no long-lived service account keys required.

```yaml
name: Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      pull-requests: write
      security-events: write # required if uploading SARIF

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
          service_account: ${{ secrets.WIF_SA }}

      - uses: OpticDiff/code-reviewer-action@v1
        with:
          model: gemini-2.5-flash
          focus: all
          min-severity: low
          sarif: results.sarif
        env:
          GOOGLE_CLOUD_PROJECT: ${{ secrets.GCP_PROJECT }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 2. Self-Hosted (vLLM / Cloud Run)

Connect to any OpenAI-compatible server such as vLLM, TGI, or a model hosted on Cloud Run:

#### vLLM / Internal Server
```yaml
- uses: OpticDiff/code-reviewer-action@v1
  with:
    model: meta-llama/Llama-3.3-70B-Instruct
    extra-args: --api-url https://vllm.internal.example.com/v1 --api-key ${{ secrets.VLLM_API_KEY }}
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### Cloud Run Endpoint
```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
    service_account: ${{ secrets.WIF_SA }}

- uses: OpticDiff/code-reviewer-action@v1
  with:
    model: custom-model
    extra-args: --api-url https://code-reviewer-proxy-xyz.a.run.app/v1
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 3. AWS Bedrock (via LiteLLM Proxy)

To use AWS Bedrock models (e.g. Anthropic Claude, Amazon Titan), point to an OpenAI-compatible proxy such as [LiteLLM](https://github.com/BerriAI/litellm):

```yaml
- uses: OpticDiff/code-reviewer-action@v1
  with:
    model: bedrock/anthropic.claude-3-5-sonnet-20241022-v2:0
    extra-args: --api-url http://litellm-proxy.internal:4000/v1 --api-key ${{ secrets.LITELLM_API_KEY }}
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Model Quality Comparison

| Model | Tier | Context | Notes |
|---|---|---|---|
| `qwen3:8b` | Demo | 32k | Local/Ollama, fast, lower quality |
| `qwen3:32b` | Good | 32k | Local/Ollama, needs 32GB RAM |
| `gemini-2.5-flash` | Recommended | 1M | Fast, excellent quality, Vertex AI |
| `gemini-2.5-pro` | Best | 1M | Deepest analysis, Vertex AI |
| `claude-sonnet-4` | Best | 200k | Excellent for code, Vertex AI |
| Any OpenAI-compat | Varies | Varies | Via `--api-url` |

---

## Action Inputs

| Input | Description | Default | Required |
|---|---|---|---|
| `version` | `code-reviewer` binary version to install from releases | `0.12.0` | No |
| `model` | Model ID to use for analysis | `gemini-2.5-flash` | No |
| `focus` | Review focus areas (`bugs`, `security`, `performance`, `style`, `docs`, `all`) | `all` | No |
| `min-severity` | Minimum severity to report (`low`, `medium`, `high`, `critical`) | `low` | No |
| `sarif` | SARIF output file path. When set, SARIF report is generated and uploaded | `""` | No |
| `platform-config` | Path, glob, or comma-separated platform config files (`.yaml`) | `""` | No |
| `platform-review-md` | Path, glob, or comma-separated platform guideline files (`.md`) | `""` | No |
| `profile` | Review profile: `platform`, `product`, or `all`. See [Dual-Review Architecture](https://github.com/OpticDiff/code-reviewer/blob/main/docs/PLATFORM-GOVERNANCE.md#dual-review-ci-architecture-platform-gate-vs-product-quality-review). | `""` | No |
| `platform-model` | Model override for `--profile=platform` (e.g. `gemini-2.5-pro`) | `""` | No |
| `product-model` | Model override for `--profile=product` (e.g. `gemini-2.5-flash`) | `""` | No |
| `platform-visibility` | Platform findings visibility: `public` or `security-team-only` | `""` | No |
| `extra-args` | Additional CLI flags passed directly to `code-reviewer` | `""` | No |

---

## Advanced Examples

### Multi-Model Consensus Review

Run multiple models in parallel and only post findings where models agree:

```yaml
- uses: OpticDiff/code-reviewer-action@v1
  with:
    extra-args: --models gemini-2.5-flash,claude-sonnet-4 --consensus-threshold 2
  env:
    GOOGLE_CLOUD_PROJECT: ${{ secrets.GCP_PROJECT }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Incremental Review & PR Description Updates

Only review files modified in the latest push, and update the PR description with a concise summary:

```yaml
- uses: OpticDiff/code-reviewer-action@v1
  with:
    model: gemini-2.5-flash
    extra-args: --incremental --update-description
  env:
    GOOGLE_CLOUD_PROJECT: ${{ secrets.GCP_PROJECT }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Enterprise Platform Governance (Central Policies)

Enforce mandatory platform rules (e.g. security mandates, monotonic severity floor) from an organization repository without letting individual repos overrule them (pass a read token if the central policy repository is private):

```yaml
- uses: actions/checkout@v4
  with:
    repository: my-org/platform-governance
    token: ${{ secrets.PLATFORM_REPO_TOKEN }} # required if private/internal
    path: .platform-central

- uses: OpticDiff/code-reviewer-action@v1
  with:
    platform-config: ".platform-central/rules/*.yaml"
    platform-review-md: ".platform-central/GUIDELINES.md"
  env:
    GOOGLE_CLOUD_PROJECT: ${{ secrets.GCP_PROJECT }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Dual-Review Architecture (Platform Gate + Product Quality)

Run two independent, isolated reviews per PR — one enforcing enterprise compliance, the other focused on code quality. Each posts its own comments and SARIF results:

```yaml
name: Dual Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  platform-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      security-events: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/checkout@v4
        with:
          repository: my-org/platform-governance
          token: ${{ secrets.PLATFORM_REPO_TOKEN }}
          path: .platform-central
      - uses: OpticDiff/code-reviewer-action@v1
        with:
          profile: platform
          platform-model: gemini-2.5-pro
          platform-config: ".platform-central/rules/*.yaml"
          platform-review-md: ".platform-central/GUIDELINES.md"
          sarif: platform.sarif
        env:
          GOOGLE_CLOUD_PROJECT: ${{ secrets.GCP_PROJECT }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  product-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      security-events: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: OpticDiff/code-reviewer-action@v1
        with:
          profile: product
          product-model: gemini-2.5-flash
          sarif: product.sarif
        env:
          GOOGLE_CLOUD_PROJECT: ${{ secrets.GCP_PROJECT }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Documentation & Source

For full CLI documentation, configuration files (`.code-reviewer.yaml`, `REVIEW.md`), and issue tracking, visit the main repository:

👉 **[https://github.com/OpticDiff/code-reviewer](https://github.com/OpticDiff/code-reviewer)**
