# claude-security-engineer
The OWASP layer for Claude Code. Security checks before your agent ships.

## The problem

AI agents generate working code. They don't generate secure code. Left unconstrained, your agent will:
- write parameterized queries wrong
- accept user-supplied URLs without an allowlist
- store passwords with SHA256
- ship without logging a single auth event
- skip rate limits on login endpoints

None of this shows up until an incident. This pack intercepts those patterns before the code is written.

## Context window and cost

Skills are lazy-loaded. A skill file sits on disk until its trigger condition is met. Most prompts load zero security skills. When triggered, each skill adds roughly 500 tokens. At current Sonnet pricing that is fractions of a cent. You control what you install.

## Skills

| Skill name | OWASP | What it checks | Trigger |
|---|---|---|---|
| `broken-access-control` | A01 | IDOR patterns, missing ownership checks, client-side auth logic | routes, endpoints, data access layers |
| `cryptographic-failures` | A02 | Password hashing algorithm, encryption at rest/transit, hardcoded secrets, weak algos | password/PII/token storage or transmission |
| `injection-check` | A03 | SQL concatenation, shell command injection, XSS, LDAP injection | queries, shell commands, HTML rendering |
| `insecure-design` | A04 | Threat model, rate limits, worst-case abuse scenario | new features involving auth, payments, data export, admin, multi-tenant |
| `security-misconfiguration` | A05 | Debug mode, CORS wildcard, security headers, cloud storage ACLs, default credentials | server config, infra code, deployment scripts |
| `vulnerable-components` | A06 | CVE status, license compatibility, maintenance health, version pinning | adding/upgrading any dependency |
| `auth-failures` | A07 | Rate limiting on login, session invalidation, JWT config, account enumeration | login, session, token, password reset, MFA |
| `data-integrity-failures` | A08 | Unsafe deserialization (pickle), CI pipeline script integrity, lockfile verification | deserialization code, CI pipelines |
| `security-logging-failures` | A09 | Structured security event logging, required fields, sensitive data in logs | auth flows, authorization checks, admin actions |
| `ssrf-check` | A10 | Allowlist validation, metadata endpoint blocking, DNS rebinding mitigation | outbound HTTP requests from user-supplied URLs |
| `owasp-security-gate` | All | Runs all 10 checks in sequence, produces a gate report | before any significant feature ships |

## What it looks like

**Without the skill:**

```
User: Add a user lookup endpoint that fetches a record by ID.

Agent: Here's the endpoint:

@app.route("/users/<int:user_id>")
@login_required
def get_user(user_id):
    user = User.query.get(user_id)
    return jsonify(user.to_dict())
```

No comment from the agent. IDOR vulnerability — any logged-in user can fetch any user's record by changing the ID.

**With broken-access-control skill:**

```
User: Add a user lookup endpoint that fetches a record by ID.

Agent:

## Access Control Check

**Object type:** User record
**Caller identity source:** Flask-Login current_user
**Authorization check present:** no
**IDOR risk:** present — user_id is user-supplied with no ownership check

### Verdict
BLOCKED — this endpoint returns any user record to any authenticated caller.
The query must filter by both user_id AND the authenticated user's id:

    user = User.query.filter_by(
        id=user_id,
        id=current_user.id
    ).first_or_404()

Adding the endpoint as written introduces a broken access control vulnerability (OWASP A01).
```

## Install

No existing `.claude` folder:

```bash
cp -r .claude /path/to/your/project/
```

Already using Claude Code:

```bash
rsync -av .claude/skills/ /path/to/your/project/.claude/skills/
```

Claude Code picks up skills from `.claude/skills/` automatically.

## Where to start

These three are the highest-signal starting point.

| Skill | Why start here |
|---|---|
| `broken-access-control` | IDOR is the most common auth vulnerability in web applications. Catches ownership check gaps on every data access route. |
| `injection-check` | SQL injection and command injection are still endemic. An agent writing database code will default to string concatenation unless told otherwise. |
| `auth-failures` | Rate limiting on login is almost never the first thing an agent adds. This makes it mandatory. |

Add the other skills as your codebase expands into each risk area. A team shipping database migrations should add `data-integrity-failures`. A team adding external integrations should add `ssrf-check`.

## Full security review

To run all 10 checks before a feature ships:

```
/owasp-security-gate
```

Produces a gate report with CLEAR / BLOCKED status for each check. Any BLOCKED item must be resolved before the feature ships.

## Built by
Swapnil Tamse — Engineering Leader, AI/AI Security, NYC
Companion to: [claude-staff-engineer](https://github.com/swapniltamse/claude-staff-engineer)

## License
Apache 2.0
