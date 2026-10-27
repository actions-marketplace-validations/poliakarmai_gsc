---
name: gsc-scan
description: Use when scanning a repo or PR with GSC and reading the coverage-aware report.
---
# GSC Scan

## When to use

You are asked to scan a repository, a local path, or a pull request diff for
vulnerabilities, secrets, dependency issues, or IaC misconfigurations with GSC,
and to interpret the resulting report. GSC is a SAST engine with PoC
verification; the engine runs from the Docker image
`ghcr.io/poliakarmai/gsc-scanner:v1` (BUSL 1.1), the Action `poliakarmai/gsc@v1`
is Apache 2.0.

## Commands

GitHub Action — one file, zero config (`.github/workflows/gsc.yml`):

```yaml
name: GSC Audit
on:
  pull_request:
    branches: [main, master]      # must match your default branch, else it never fires
jobs:
  audit:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: poliakarmai/gsc@v1
        with:
          fail_on_critical: false
          # fail_on_score: 80      # fail the PR when the score drops below this
          # sarif: true            # also emit SARIF for GitHub Code Scanning
```

CLI in Docker:

```bash
docker run --rm -v "$PWD:/repo:ro" -w /repo \
  ghcr.io/poliakarmai/gsc-scanner:v1 scan . --ci --json
```

Useful Action inputs: `path` (default `.`), `deep_scan` (default `true`),
`fail_on_critical`, `fail_on_score` (0 = off, 1–100), `max_findings_to_comment`
(default 10), `sarif`, `reachability`, and BYO-LLM `llm_api_key` / `llm_base_url` /
`llm_model` (without a key the engine runs regex-only).

## How to read the result

1. Read `scan_status` first: `ok` or `partial`.
2. Read the coverage block before trusting any verdict:
   - `files_scanned / files_total / files_truncated` — how much of the tree was seen;
   - `detectors_total / detectors_ran / detectors_not_run` — which detectors executed.
3. `scan_status: partial` (a cap hit, not a crash) means a low finding count is
   **not** a clean repo. Say which detectors did not run.
4. Per finding, check whether a PoC is attached. Two layers: a deterministic
   template (works without an LLM key, no confidence gate) and an LLM PoC for
   findings with `confidence >= 0.80`.
5. `verify_finding` answers `status: "unavailable"` when the sandbox cannot start
   or no PoC is attached. That is **not** a verdict about the finding.

## Pitfalls

- Do not report "clean" on an empty finding list while `detectors_not_run` is
  non-empty — that is an incomplete scan, not a pass.
- `pr-gate` / diff mode only sees changed files; it can never give a whole-repo
  verdict. Use a full scan for that.
- `scan_repo` is bounded by `GSC_SCAN_TIMEOUT` (default 300 s) and
  `GSC_MAX_FILES` (default 50 000). A truncated scan is not persisted.
- No PoC attached ≠ false positive: several classes (e.g. hardcoded secrets) have
  no deterministic template, and `UNAVAILABLE` is the honest answer.
