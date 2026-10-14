# Security Policy

## Supported Versions

| Version | Supported |
| ------- | --------- |
| v1.x    | ✅        |

## Reporting a Vulnerability

We take security issues seriously. If you discover a vulnerability in GSC
(the GitHub Action, the MCP server, or the Docker engine image), please report
it privately:

- **GitHub Private Vulnerability Reporting** (preferred):
  https://github.com/poliakarmai/gsc/security/advisories/new
- **Email:** rgtsystems96@gmail.com

**Please do not open a public issue for security problems.**

### What to include

- Affected component and version (Action `v1.x`, engine image tag, MCP server)
- Reproduction steps or a minimal proof of concept
- Impact assessment (what an attacker could do)
- Any suggested fix (optional)

### What to expect

- **Acknowledgement** within 3 business days
- **Initial assessment** within 10 business days
- **Fix and coordinated disclosure** — we aim to release a patch within 30 days
  of confirming a valid report, and will credit reporters who wish to be named
  in `HALL_OF_FAME.md`

## Scope

**In scope:** GSC GitHub Action, MCP server (`gsc_cloud/gsc_mcp_server.py`),
Docker engine image, and the public repository contents.

**Out of scope:** findings produced by scanning third-party repositories
(those belong to the upstream project), and issues in dependencies that are
already covered by our SCA/EPSS pipeline.

## Safe Harbor

We will not pursue legal action against researchers who act in good faith,
avoid privacy violations and service disruption, and give us reasonable time
to remediate before public disclosure.
