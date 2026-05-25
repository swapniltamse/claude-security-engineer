---
name: vulnerable-components
description: Checks CVE status, license compatibility, and maintenance health of any new dependency before it enters the codebase.
when_to_use: Apply automatically when adding, upgrading, or pinning any package, library, or external dependency to the project.
---

# Vulnerable Components Skill

The most common way to ship a known vulnerability is to add a package without checking whether it has one. An agent adding a dependency installs the version that satisfies the constraint and moves on. It does not check CVEs. It does not check whether the package is actively maintained. It does not check whether the license conflicts with your distribution terms.

## What to Check Before Adding or Upgrading a Dependency

**1. CVE check**
Before adding or upgrading a package, verify the specific version has no critical or high CVEs. Use: `pip-audit`, `npm audit`, `snyk test`, `grype`, or check the package's advisory page directly.

**2. License compatibility**
Identify the license of the package being added. Flag GPL licenses in proprietary codebases — GPL has copyleft requirements that can affect distribution. MIT, Apache 2.0, and BSD are generally safe. Unknown or custom licenses require legal review.

**3. Maintenance health**
A package with no commits in 24 months, no response to open issues, or fewer than 100 stars for a critical dependency is a risk. Flag unmaintained packages.

**4. Transitive dependencies**
Run `pip-audit`, `npm audit`, or `cargo audit` against the full dependency tree, not just the direct addition. A clean package with a vulnerable transitive dependency is still vulnerable.

**5. Version pinning**
Dependencies must be pinned to a specific version in production, not floating ranges like `^1.2` or `>=2.0`. Unpinned dependencies can update automatically to a vulnerable version.

## Output Format

```
## Vulnerable Components Check

**Package:** [name@version]
**Language/ecosystem:** [Python / Node / Go / Rust / Java / other]

### Findings
- CVE status (this version): [no known CVEs / list CVEs found — BLOCKED if critical/high]
- License: [license name — flag if GPL or unknown]
- Last commit: [date — flag if > 24 months ago]
- Open issues/PRs: [number — flag if > 500 stale with no maintainer activity]
- Transitive audit: [clean / vulnerabilities found — list]
- Version pinned: [yes / no — BLOCKED if no]

### Verdict
[CLEAR / BLOCKED — list each risk]
```

## Hard Stops

Block before adding or upgrading a dependency if:

- The package version has a critical or high CVE with no available patch
- The license is GPL and the project is proprietary or commercially distributed
- The package has no commit activity in the last 24 months and is used in a security-sensitive path
- Transitive dependency audit finds a critical vulnerability
- The version is not pinned in production configuration

## What Good Looks Like

```
# Good: pinned version with audit clean
requests==2.31.0  # pip-audit clean as of 2026-05-25

# Bad: floating range
requests>=2.20
```

Run before every dependency addition:
```bash
pip install package && pip-audit
# or
npm install package && npm audit
```
