---
name: gsc-triage
description: Use when triaging GSC findings — blocking set, TP vs noise, PoC verification.
---
# GSC Triage

## When to use

You have GSC findings (from the Action, the CLI, or MCP) and must decide what
blocks a merge, what is noise, and which findings a PoC actually confirms.

## Commands

MCP tools (server `gsc`, shipped inside the engine image):

```
list_findings     # recent findings from the local DB, no scan
get_finding       # full detail for one finding_key
get_poc           # the PoC attached to a finding (DB or deterministic template)
verify_finding    # re-run one PoC in the sandbox
verify_findings   # same for a batch of keys
doctor            # self-check: sandbox, state volume, allowed roots, cache, lock
list_detectors    # which detectors the engine has
```

CLI context helpers:

```bash
docker run --rm -v "$PWD:/repo:ro" -w /repo ghcr.io/poliakarmai/gsc-scanner:v1 sca --repo .
docker run --rm ghcr.io/poliakarmai/gsc-scanner:v1 epss --cve CVE-XXXX-YYYY
```

## How to read the result

Triage in this order:

1. **Coverage first.** If `detectors_not_run` is non-empty or `scan_status` is
   `partial`, the finding set is incomplete — say so before discussing findings.
2. **Blocking set.** The Action decides it via `fail_on_critical` and
   `fail_on_score` (threshold 1–100). Findings that make the gate exit non-zero
   are the blocking subset; everything else is advisory.
3. **Verified = confirmed.** `verify_finding` returning an exploit status means
   the PoC reproduced — treat the finding as confirmed and fix or explicitly
   accept it.
4. **`unavailable` is not "not reproducible".** It means the sandbox could not
   start (missing user-namespace flags, no PoC attached, wrong volume). Run
   `doctor` before drawing any conclusion.
5. **No PoC ≠ false positive.** Secrets in particular have no deterministic
   template; a live credential stays a true positive without any PoC.
6. **`inconclusive` stays open.** Report it as "static match, unverified" rather
   than dismissing it.

## Pitfalls

- `list_findings` reads the DB, not your working tree: confirm it is the same
  repo run you are triaging (`get_finding` re-scans only when the key is absent).
- Batch with `verify_findings` instead of looping `verify_finding`.
- `verify_fix` with status `fixed` means "the detector no longer fires after a
  re-scan" (GSC-003), not "the exploit was executed and failed". Do not overclaim.
- DESTRUCTIVE and non-deterministic PoCs are refused by design; a refusal is a
  policy verdict, not a statement about the vulnerability.
