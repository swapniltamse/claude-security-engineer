---
name: security-logging-failures
description: Requires structured security event logging before any authentication, authorization, or sensitive data access code ships.
when_to_use: Apply automatically when writing or modifying authentication flows, authorization checks, admin functions, data export, payment processing, or any operation where a security-relevant event should be auditable.
---

# Security Logging and Monitoring Failures Skill

Security logging failures mean that when an attack happens, nobody can tell. No audit trail. No structured events. Logs that exist but contain no useful fields. An agent writing authentication or data access code produces code that works. It does not produce code that can be investigated after a breach.

## What to Log Before Shipping Security-Sensitive Code

Every security-sensitive operation must emit a structured log event with these fields at minimum:

- `timestamp` — ISO 8601, UTC
- `event_type` — machine-readable category (`auth.login.success`, `auth.login.failure`, `access.denied`, `data.export`, `admin.action`)
- `user_id` — the authenticated identity, or `anonymous` if not authenticated
- `ip_address` — source IP of the request
- `resource` — what was accessed or attempted
- `result` — `success` or `failure`
- `reason` — for failures, a specific reason (not just "error")

Security events that require logging:

- Login success and failure
- Logout
- Password reset request and completion
- MFA success and failure
- Access denied (authorization check failed)
- Admin actions (any privileged operation)
- Sensitive data access (PII export, bulk download)
- Account creation and deletion
- Permission changes

## What Must NOT Appear in Logs

- Passwords or password hashes
- Full credit card numbers or CVVs
- Social Security Numbers or government IDs
- API keys or tokens (log the last 4 characters only)
- Session tokens

## Output Format

Before writing security-sensitive code, confirm the logging plan:

```
## Security Logging Check

**Operation:** [what this code does]
**Security events generated:** [list each event type]

### Logging Plan
For each event:
- Event type: [machine-readable name]
- Fields logged: [timestamp, user_id, ip, resource, result, reason]
- Sensitive fields excluded: [confirm none of the blocked fields appear]

### Verdict
[CLEAR — logging plan complete / BLOCKED — list missing event types or fields]
```

## Hard Stops

Block before shipping if:

- Authentication events (success or failure) produce no structured log
- Access denied events are not logged
- A log entry contains a password, full token, or government ID
- Logs are unstructured plain text with no queryable fields
- Admin actions produce no audit record

## What Good Looks Like

```python
# Good: structured security event
import structlog
log = structlog.get_logger()

def login(email, password, ip_address):
    user = authenticate(email, password)
    if user:
        log.info("auth.login.success",
            user_id=user.id,
            ip_address=ip_address,
            timestamp=datetime.utcnow().isoformat()
        )
    else:
        log.warning("auth.login.failure",
            user_id=None,
            ip_address=ip_address,
            reason="invalid_credentials",
            timestamp=datetime.utcnow().isoformat()
        )
```
