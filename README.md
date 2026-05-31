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

## Philosophy

Most security tools in this space run after the code is written. They scan what exists, produce a report, and hand it back. The code already has a shape. The patterns are already set. Findings become negotiation.

This pack works the opposite way.

The parallel is test-driven development. In TDD, you write the test before the implementation. The test defines what "correct" looks like before any code exists to be defensive about. The result is code written to satisfy a constraint, not code retrofitted to pass a check.

These skills apply the same logic to security. Each skill runs before your agent writes anything in that risk area. There is no code yet. No patterns to defend. The agent knows the constraint before it makes a single design decision.

That is not a subtle difference. It changes what gets built.

**What about putting OWASP in the requirements spec?**

That works at project start on a greenfield build. It breaks down in two common situations: existing codebases where you're adding features mid-project, and long sessions where the agent has moved far from the original spec by the time it's writing the tenth endpoint. "Ensure OWASP compliance" in a requirements doc is aspiration. A skill that fires before writing a specific route asks a specific question: does this query verify the caller owns the record before returning it? There is a real difference between a general reminder from hour one and a targeted check at the moment of relevance. That gap between aspiration and specific check is where incidents happen.

[`agamm/claude-code-owasp`](https://github.com/agamm/claude-code-owasp) packages OWASP Top 10 as a single skill you invoke after writing. [`trailofbits/skills`](https://github.com/trailofbits/skills) is a professional audit toolkit for security researchers. Both are useful. Neither is designed to intercept the agent before it starts.

This pack is.

## How to use

Each skill is a slash command. Type it before you ask the agent to write the relevant code.

```
/broken-access-control
Add a user profile endpoint that fetches a record by ID.
```

The skill loads into the conversation and the agent runs its check before writing anything.

You can also invoke skills mid-conversation if you realize you're about to touch a sensitive area:

```
/injection-check
Now add the search filter to that query.
```

To run all 10 checks at once before shipping a feature:

```
/owasp-security-gate
```

## Skills

Each skill maps to one OWASP Top 10 category. Invoke it when you're about to write code in that area.

| Skill | Invoke when you're about to... | OWASP |
|---|---|---|
| `/broken-access-control` | Write a route, endpoint, or query that fetches user-specific data | A01 |
| `/cryptographic-failures` | Store passwords, tokens, or PII — or transmit sensitive data | A02 |
| `/injection-check` | Write a database query, shell command, or render user input as HTML | A03 |
| `/insecure-design` | Add a new feature touching auth, payments, data export, admin, or multi-tenant | A04 |
| `/security-misconfiguration` | Touch server config, environment variables, CORS, headers, or cloud storage | A05 |
| `/vulnerable-components` | Add or upgrade a dependency | A06 |
| `/auth-failures` | Build login, logout, session handling, password reset, or MFA | A07 |
| `/data-integrity-failures` | Deserialize external data, modify a CI pipeline, or change lockfiles | A08 |
| `/security-logging-failures` | Write auth flows, authorization checks, or admin actions | A09 |
| `/ssrf-check` | Make an outbound HTTP request using a URL from user input | A10 |
| `/owasp-security-gate` | Before any significant feature ships | All |

## Context window and cost

Skills are lazy-loaded. A skill file sits on disk until you invoke it. Most prompts load zero security skills. When invoked, each skill adds roughly 500 tokens. At current Sonnet pricing that is fractions of a cent. You control what you install.

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
User: /broken-access-control
      Add a user lookup endpoint that fetches a record by ID.

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
        owner_id=current_user.id
    ).first_or_404()

Adding the endpoint as written introduces a broken access control vulnerability (OWASP A01).
```

## Overriding a BLOCKED verdict

A BLOCKED verdict is a checkpoint, not a hard lock. The agent surfaces the risk and waits. You decide.

To override, tell the agent explicitly what you know:

```
The user table has no owner_id column — this is an internal admin endpoint,
only reachable by service accounts. Proceed with the original query.
```

The agent will document your rationale and continue. This keeps the override visible in the conversation log rather than silently bypassing the check.

False positives are expected in test and local development contexts. The skills are designed for pre-ship review, not every prompt. If a skill fires too aggressively in your workflow, remove it from `.claude/skills/` in that project.

## Install

Skills install per project. They live in the project's `.claude/skills/` directory and do not affect your global Claude Code setup or any other project.

Run this in your project directory:

**Mac / Linux:**
```bash
git clone https://github.com/swapniltamse/claude-security-engineer /tmp/cse && mkdir -p .claude/skills && cp -r /tmp/cse/.claude/skills/. .claude/skills/
```

**Windows (PowerShell):**
```powershell
git clone https://github.com/swapniltamse/claude-security-engineer $env:TEMP\cse; mkdir -Force .claude\skills; Copy-Item -Recurse $env:TEMP\cse\.claude\skills\* .claude\skills\
```

Merges into your existing `.claude` setup without overwriting anything.

## Where to start

These three are the highest-signal starting point.

| Skill | Why start here |
|---|---|
| `broken-access-control` | IDOR is the most common auth vulnerability in web applications. Catches ownership check gaps on every data access route. |
| `injection-check` | SQL injection and command injection are still endemic. An agent writing database code will default to string concatenation unless told otherwise. |
| `auth-failures` | Rate limiting on login is almost never the first thing an agent adds. This makes it mandatory. |

Add the other skills as your codebase expands into each risk area. A team shipping database migrations should add `data-integrity-failures`. A team adding external integrations should add `ssrf-check`.

## Full security review

Run `/owasp-security-gate` before any significant feature ships. It runs all 10 checks in sequence and produces a gate report with CLEAR / BLOCKED status per category. Any BLOCKED item must be resolved before the feature ships.

## Built by
[Swapnil Tamse](https://swapniltamse.com) — Engineering Leader, AI/AI Security, NYC
[LinkedIn](https://www.linkedin.com/in/swapniltamse)
Companion to: [claude-staff-engineer](https://github.com/swapniltamse/claude-staff-engineer)

## License
Apache 2.0
