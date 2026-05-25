---
name: auth-failures
description: Enforces secure authentication patterns before any login, session, token, or credential management code is written or modified.
when_to_use: Apply automatically when writing or modifying authentication flows, login endpoints, session management, token issuance, password reset, MFA, or credential storage.
---

# Authentication Failures Skill

Authentication failures include broken login flows, weak session management, and credential handling that is technically functional but exploitable. An agent implementing an auth flow will produce something that works. The question is whether it resists brute force, session fixation, credential stuffing, and token leakage.

## What to Check Before Writing Auth Code

**1. Rate limiting on login and credential endpoints**
Login, password reset, and MFA verification endpoints must have rate limiting. Without it, brute force and credential stuffing are possible. State the limit: requests per minute, lockout threshold, and lockout duration.

**2. Session management**
Sessions must be invalidated on logout. Session tokens must be regenerated after privilege escalation (e.g., after login, after MFA completion). Session tokens must not appear in URLs.

**3. Token security**
JWTs must use a strong algorithm (RS256, ES256, HS256 with a secret of 32+ bytes). The `alg: none` attack must be blocked. JWT secrets must not be hardcoded. Token expiry must be set.

**4. Password policy**
Minimum length of 8 characters. No maximum length below 64 characters (password truncation is a vulnerability). Rejection of known breached passwords (Have I Been Pwned list) is best practice.

**5. MFA**
For any system handling financial data, health data, or admin access, MFA is required. State whether it is implemented, planned, or explicitly out of scope with justification.

**6. Account enumeration**
Login failure messages must not distinguish between "user not found" and "wrong password." Both conditions must return the same response.

## Output Format

```
## Authentication Failures Check

**Auth flow type:** [login / registration / password reset / session / token / MFA]

### Findings
- Rate limiting: [yes — state limit / no — BLOCKED]
- Session invalidation on logout: [yes / no — BLOCKED / not applicable]
- Token algorithm: [algorithm name — flag if alg:none or weak symmetric]
- Token expiry set: [yes / no — BLOCKED if no]
- Account enumeration prevention: [yes / no]
- MFA: [implemented / planned / out of scope — justification required if out of scope for sensitive systems]

### Verdict
[CLEAR / BLOCKED — list each failure]
```

## Hard Stops

Block before writing auth code if:

- A login or password reset endpoint has no rate limit
- Session tokens are not invalidated on logout
- JWT accepts `alg: none` or uses a weak/hardcoded secret
- Token expiry is absent or set to more than 24 hours without a refresh strategy
- Login error messages reveal whether the username or email exists in the system

## What Good Looks Like

```python
# Good: rate-limited login with account enumeration prevention
@limiter.limit("5/minute")
@app.route("/login", methods=["POST"])
def login():
    user = User.query.filter_by(email=request.form["email"]).first()
    if not user or not user.check_password(request.form["password"]):
        return jsonify({"error": "Invalid credentials"}), 401  # same message both cases
    session.clear()         # session fixation prevention
    session["user_id"] = user.id
    return jsonify({"status": "ok"})
```
