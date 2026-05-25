---
name: owasp-security-gate
description: Full pre-ship security review that runs all ten OWASP Top 10 checks in sequence. Use before any significant feature ships to production.
when_to_use: Use explicitly with /owasp-security-gate before merging a feature that touches authentication, data access, external requests, or configuration. Also use when asked to do a full security review.
---

# OWASP Security Gate (Meta-Skill)

This skill runs all ten OWASP Top 10 (2021) checks in sequence. Use it before a feature ships, before a PR is marked ready for review, or any time a full security review is needed.

Each check below maps to a dedicated skill. Run them in order. Mark each CLEAR or BLOCKED. If any check is BLOCKED, the feature does not ship until it is resolved.

## Run in Sequence

### A01 — Broken Access Control
Apply `broken-access-control` skill.
Trigger: any code that retrieves, modifies, or deletes user data.

Key questions:
- Is object-level authorization enforced? (No IDOR)
- Is the access control check server-side?
- Does access default to deny?

### A02 — Cryptographic Failures
Apply `cryptographic-failures` skill.
Trigger: any code that stores or transmits passwords, tokens, or PII.

Key questions:
- Are passwords hashed with bcrypt, scrypt, Argon2, or PBKDF2?
- Is sensitive data encrypted at rest and in transit?
- Are there hardcoded secrets?

### A03 — Injection
Apply `injection-check` skill.
Trigger: any code that incorporates user input into SQL, shell commands, HTML, or LDAP.

Key questions:
- Are all SQL queries parameterized?
- Does any code pass user input to a shell?
- Is HTML output escaped?

### A04 — Insecure Design
Apply `insecure-design` skill.
Trigger: any new user-facing feature, auth flow, or data handling system.

Key questions:
- Is there a threat model?
- Are rate limits defined?
- Is the worst-case abuse scenario documented?

### A05 — Security Misconfiguration
Apply `security-misconfiguration` skill.
Trigger: any configuration, infrastructure, or deployment code.

Key questions:
- Is debug mode off?
- Is CORS restricted to specific origins?
- Are security headers present?

### A06 — Vulnerable Components
Apply `vulnerable-components` skill.
Trigger: any new or upgraded dependency.

Key questions:
- Are there known CVEs in this version?
- Is the license compatible?
- Is the dependency pinned?

### A07 — Authentication Failures
Apply `auth-failures` skill.
Trigger: any login, session, token, or credential code.

Key questions:
- Is there rate limiting on login?
- Are sessions invalidated on logout?
- Is JWT configured securely?

### A08 — Data Integrity Failures
Apply `data-integrity-failures` skill.
Trigger: any deserialization, CI pipeline, or update mechanism code.

Key questions:
- Is untrusted data deserialized with pickle or equivalent?
- Does the CI pipeline use pinned dependencies?

### A09 — Security Logging Failures
Apply `security-logging-failures` skill.
Trigger: any authentication, authorization, or sensitive data access code.

Key questions:
- Are auth events logged with structured fields?
- Do logs contain any sensitive data (passwords, tokens)?

### A10 — SSRF
Apply `ssrf-check` skill.
Trigger: any code that fetches a URL based on user input.

Key questions:
- Is there an allowlist of permitted hosts?
- Is the AWS metadata endpoint blocked?

## Summary Report Format

After running all ten checks:

```
## OWASP Security Gate Report

**Feature:** [name]
**Date:** [YYYY-MM-DD]

| Check | OWASP | Status | Blocker |
|---|---|---|---|
| broken-access-control | A01 | CLEAR / BLOCKED | [issue if blocked] |
| cryptographic-failures | A02 | CLEAR / BLOCKED | |
| injection-check | A03 | CLEAR / BLOCKED | |
| insecure-design | A04 | CLEAR / BLOCKED | |
| security-misconfiguration | A05 | CLEAR / BLOCKED | |
| vulnerable-components | A06 | CLEAR / BLOCKED | |
| auth-failures | A07 | CLEAR / BLOCKED | |
| data-integrity-failures | A08 | CLEAR / BLOCKED | |
| security-logging-failures | A09 | CLEAR / BLOCKED | |
| ssrf-check | A10 | CLEAR / BLOCKED | |

### Overall Verdict
[ALL CLEAR — feature is ready to ship]
[BLOCKED — [N] checks require resolution before shipping]
```
