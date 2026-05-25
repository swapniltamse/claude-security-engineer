---
name: insecure-design
description: Requires a threat model before any new user-facing feature, authentication flow, or data handling system is designed or modified.
when_to_use: Apply automatically before designing or implementing a new feature involving user authentication, payment flows, data export, admin functions, or multi-tenant isolation.
---

# Insecure Design Skill

Insecure design means the vulnerability is in the architecture, not the implementation. No amount of code review catches a design that allows users to reset other users' passwords, a multi-tenant system that shares data incorrectly, or an API that exposes bulk export without rate limiting. These are design decisions. They need to be examined before code is written.

## Required Before Starting Implementation

For any new feature in scope (auth flows, payment handling, admin functions, data export, multi-tenant boundaries, bulk operations), produce a threat model with these four questions answered:

**1. What is the trust boundary?**
Where does authenticated context begin and end? What does the system trust that it should not? Who can call this endpoint or function, and what can they do?

**2. What is the worst-case abuse scenario?**
If an attacker has a valid account, what is the most damaging thing they can do with this feature? If the system has no rate limit, what happens when someone calls this endpoint 10,000 times? If an ID is sequential, what happens when someone increments it?

**3. What data is handled and where does it go?**
Map the data: user input arrives, what gets persisted, what gets returned, what gets logged, what gets sent to a third party. Any step where sensitive data is broader than necessary is a design flaw.

**4. Is there a rate limit, retry limit, or volume constraint?**
Bulk operations, password resets, login attempts, and API calls without limits are design vulnerabilities. State the limit or explain why one is not required.

## Output Format

```
## Insecure Design Review

**Feature:** [name and one-sentence description]
**Scope:** [authentication / payment / data export / admin / multi-tenant / other]

### Threat Model
**Trust boundary:** [where trust is established and where it ends]
**Worst-case abuse scenario:** [specific attack description, not "an attacker could do bad things"]
**Data flow:** [input -> processing -> persistence -> output -> third parties]
**Rate/volume limits:** [what limits exist or why none are needed]

### Design Risks Found
[List each risk. Be specific about the mechanism, not just the category.]

### Verdict
[CLEAR — proceed with implementation / BLOCKED — list design changes required before coding]
```

## Hard Stops

Block before writing code if:

- A password reset or account recovery flow has no rate limit on attempts
- A multi-tenant feature has no documented isolation mechanism
- A bulk data export has no volume or rate constraint
- An admin function is accessible based on a client-supplied parameter rather than a server-side role check
- The worst-case abuse scenario includes privilege escalation with no documented mitigation

## What Good Looks Like

A feature that earns CLEAR has:
1. A documented trust boundary
2. A named worst-case abuse scenario with a specific mitigation for each
3. Rate limits defined before implementation starts
4. Data scoped to the minimum necessary for the feature to work
