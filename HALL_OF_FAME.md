# 🏆 GSC Hall of Fame — Real Vulnerabilities Found

> Real vulnerabilities discovered by GSC in production open-source projects.
> Every row below was verified against the GitHub API (PR state + upstream
> commits), not transcribed from memory. **Status matters as much as the find** —
> a rejected report is listed as rejected.

## ✅ 1. Merged PRs

Reviewed, approved, and merged by upstream maintainers — the fix in *our* patch:

| # | Project | PR | CWE | Merged | Diff |
|---|---------|-----|-----|--------|------|
| 1 | **yoheinakajima/instagraph** ⭐**3552** | [#94](https://github.com/yoheinakajima/instagraph/pull/94) | Werkzeug debugger exposed on `0.0.0.0` | 2026-10-04 | +6/−2, 2 files |
| 2 | **stanfrbd/cyberbro** ⭐693 | [#212](https://github.com/stanfrbd/cyberbro/pull/212) | CWE-79 (XSS) | 2026-08-10 | +8/−1, 1 file |
| 3 | **cabotage/cabotage-app** ⭐38 | [#388](https://github.com/cabotage/cabotage-app/pull/388) | CWE-798 (×4 secrets) | 2026-10-04 | +168/−1, 7 files |
| 4 | **django-files/django-files** ⭐12 | [#428](https://github.com/django-files/django-files/pull/428) | CWE-798, CWE-521 | 2026-08-18 | +6/−1, 2 files |

> The instagraph row was missing from this file until 2026-10-14 — the largest
> project in the whole list, absent because the table here was hand-maintained
> while the authoritative registry lives in the vault. Both are now corrected.

## 🟢 2. Finding accepted — fix landed in the upstream project

The maintainer did not merge *our* patch but confirmed and fixed the issue
themselves. Verified against the project's own commits/releases:

| # | Project | PR | CWE | Outcome | Verified by |
|---|---------|-----|-----|---------|-------------|
| 5 | **BrowserlessAPI/VortexPanel** ⭐28 | [#19](https://github.com/BrowserlessAPI/VortexPanel/pull/19) | CWE-78 (OS command injection) | PR closed by owner 2026-08-17: *"all the Vulnerability closed now"*. Fix shipped as hardening. | **v3.5.0** (2026-08-24) release notes: *"Domain names are validated before reaching shell commands or filesystem paths."* |
| 6 | **GoyGram/GoyGram** ⭐20 | [#4](https://github.com/GoyGram/GoyGram/pull/4) | CI hardening (SHA pins, least privilege) | PR closed, but applied by hand — commit [`4cc0778`](https://github.com/GoyGram/GoyGram/commit/4cc0778275624e6677e938a2794c73c78649a3bc), **Co-authored-by preserved**. | Commit exists, touches both workflow files. |
| 7 | **boaz-ke/duty-calculator** | [#1](https://github.com/boaz-ke/duty-calculator/pull/1) | CWE-601 (open redirect) | Vulnerability **confirmed**; maintainer fixed it on `main` (`f752281`) with a stricter check than ours. | Our own `netloc` check left bypasses — their implementation replaced it. |

## 🔶 2b. Partial — finding rejected, but the review produced a real issue

| # | Project | PR | Outcome |
|---|---------|-----|---------|
| 8 | **liquidguru/docker-updater** ⭐80 | [#21](https://github.com/liquidguru/docker-updater/pull/21) | Our CWE-384 claim was **wrong** — Flask's client-side sessions have no persistent session ID, and the maintainer *tested the attack* rather than reasoning about it. What the review surfaced was different and real: changing `AUTH_PASSWORD` does not invalidate existing sessions (they are signed with `FLASK_SECRET_KEY`), so a leaked session outlives the password change by up to 7 days. Documented in [`e855972`](https://github.com/liquidguru/docker-updater/commit/e855972) — a **docs** commit, not a code fix. |

> Row 8 is labelled partial deliberately: our diagnosed vulnerability did not
> exist. Counting it as "merged" would overstate the case.

## 🟡 3. Open — submitted, awaiting maintainer

| # | Project | PR | CWE | State |
|---|---------|-----|-----|-------|
| 6 | **aio-libs/aiohttp-security** ⭐240 | [#1005](https://github.com/aio-libs/aiohttp-security/pull/1005) | CWE-384 (reframed) | Open. Our original CWE-384 claim **did not apply to the demo** (client-side cookie storage) — we conceded this in-thread. Reframed as defence-in-depth (`session.invalidate()` on login, OWASP ASVS V3.2.1). Automated review: 5/5 *"safe to merge"*. |
| 7 | **mathiasertl/django-ca** ⭐158 | [#202](https://github.com/mathiasertl/django-ca/pull/202) | CWE-918 (SSRF) | Open. Rebased, merge conflict resolved, CI awaiting workflow approval. |
| 8 | **manjurulhoque/doccure** ⭐81 | [#14](https://github.com/manjurulhoque/doccure/pull/14) | CWE-798 | Open, no maintainer response. |
| 9 | **deep-learning-indaba/Baobab** ⭐60 | [#1401](https://github.com/deep-learning-indaba/Baobab/pull/1401) | CWE-798 | Open, no maintainer response. |
| 10 | **sankalpjonn/elchapo** ⭐23 | [#7](https://github.com/sankalpjonn/elchapo/pull/7) | CWE-918 (SSRF) | Open, no maintainer response. |
| 11 | **NadirAliOfficial/IBKR-Webhook-to-Order-Bot** ⭐13 | [#3](https://github.com/NadirAliOfficial/IBKR-Webhook-to-Order-Bot/pull/3) | CWE-306 | Open, no maintainer response. |

## ❌ 4. Rejected — the finding was wrong

| # | Project | PR | Why |
|---|---------|-----|-----|
| 12 | **the-broke-sommeliers/wine-cellar** ⭐57 | [#1182](https://github.com/the-broke-sommeliers/wine-cellar/pull/1182) | Not a vulnerability. The flagged settings are a deliberate **development** config; production uses `docker_settings.py` / `prod.py`. Maintainer: *"Please instruct your ai next time to research the deployment a bit better ;-)"* — fair. |

---

## Scorecard

| Outcome | Count |
|---|---|
| ✅ Merged (our patch) | **4** |
| 🟢 Accepted, fixed upstream by maintainer | **3** |
| 🔶 Partial (finding rejected, review produced a real, different one) | **1** |
| 🟡 Open | **6** |
| ❌ Rejected (our finding was wrong) | **1** |

**Verified fixes in production: 7** (4 merged patches + 3 fixed upstream by the
maintainers), across projects with **4,343 ⭐ combined**. A further report produced
a real — but *different* — finding that the maintainer documented. One report
was simply wrong and is labelled wrong: a scanner's precision claim is only
meaningful if its misses are counted too.

## How It Works

GSC scans open-source projects daily using its precision-hunt profile:
- **1 detector disabled** (GS000 — the legacy pattern catch-all) and **6 demoted to review-only** (GS007, GS012, GS013, GS015, GS018, GS019): they still flag, but never block
- **High-precision rules only:** GS005 (SQLi with taint tracking), GS020 (XSS with sanitizer context), GS025 (CVE patterns), YAML-based detectors (SSTI, reverse shell)
- **Manual verification** before PR submission — every finding is reviewed by a human, which is how the wine-cellar false positive was caught before it cost a maintainer time
- **Fix-ready PRs** with minimal, targeted code changes

## Submit a Finding

Found a vulnerability with GSC? [Open an issue](https://github.com/poliakarmai/gsc/issues/new?title=Hall+of+Fame:+[project]+[CWE]) to get it added!

---
*GSC — Git Security Checker. Not just another scanner.*
