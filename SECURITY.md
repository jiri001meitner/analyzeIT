# Security Policy

## Supported versions

Security fixes target the current `main` branch. Older commits and
unmaintained branches are not supported.

## Reporting a vulnerability

**Do not disclose an unpatched vulnerability in a public issue, pull request,
discussion, or commit.**

Report privately to the maintainer at `jiri@meitner.cz`.

Include the affected component and version or commit, reproduction steps
using synthetic data, required privileges, and potential impact.
Relevant findings include sandbox escapes, privacy failures, authentication
bypasses, or secret leaks.
Do not include production credentials or personal data in the report.

## Coordinated disclosure

Coordinate remediation and publication with the maintainer. Publish details
only after a fix or mitigation is available to affected users.

Security design and operational documentation:
[ARCHITECTURE.md](ARCHITECTURE.md#security-boundaries-and-references).
