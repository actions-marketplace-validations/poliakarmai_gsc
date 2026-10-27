---
name: gsc-fix-verify
description: Use when fixing a GSC finding and proving it with verify_fix and a re-scan.
---
# GSC Fix & Verify

## When to use

You are asked to fix a specific GSC finding or the blocking set, and then to
prove the fix — not merely to make the finding disappear from the report.

## Commands

```
# 1. Inspect
get_finding      # location, class, evidence, finding_key
get_poc          # the reproduction: deterministic template or LLM PoC

# 2. Edit the code so the reproduction no longer works.

# 3. Prove
verify_fix       # re-scan after the patch: fixed | still_vulnerable | inconclusive
verify_finding   # fallback when verify_fix has nothing to compare for this class

# 4. Confirm no regression (whole repo, not just the patch)
docker run --rm -v "$PWD:/repo:ro" -w /repo \
  ghcr.io/poliakarmai/gsc-scanner:v1 scan . --ci --json
```

In CI the operational definition of done is: the Action step exits zero with
your configured `fail_on_critical` / `fail_on_score`.

## How to read the result

- `verify_fix: fixed` = the detector no longer fires on a re-scan (GSC-003).
  That is a re-scan verdict, **not** an executed-exploit verdict — say it that way.
- `verify_fix: still_vulnerable` = the finding survives; do not claim the fix landed.
- `verify_fix: inconclusive` / `unavailable` = no evidence either way. Check
  `doctor` (sandbox, state volume, allowed roots) and re-run; do not convert it
  into "fixed" by wishful reading.
- After the verdict, re-check the report: the finding is gone, no new finding of
  the same class appeared elsewhere (common when a fix is copy-pasted), and
  `detectors_not_run` is empty or explicitly noted as a residual blind spot.

## Pitfalls

- Fixing the pattern without fixing the behaviour (base64-ing a secret instead of
  loading it from config) passes the detector and leaves the bug. Verify
  semantics, not just the disappearing finding.
- For credentials, "fixed" means removed **and** rotated — removing the string
  from a file does not un-leak a key that was committed.
- A green diff-scoped gate (`scan_diff`) is not a green repo: re-scan fully after
  a fix.
- `--read-only` on the state volume is unsupported; findings would come back with
  `persisted: false` and `verify_fix` would have nothing to compare.
