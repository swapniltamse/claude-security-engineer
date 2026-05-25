---
name: broken-access-control
description: Checks authorization enforcement before any route, endpoint, or data access code is written or modified. Catches missing auth checks, IDOR patterns, and privilege escalation paths.
when_to_use: Apply automatically when writing or modifying routes, API endpoints, data access layers, admin functions, or any code that retrieves or modifies user-specific data.
---

# Broken Access Control Skill

Broken access control is the number one OWASP risk category. It means the application allows users to act outside their intended permissions. An agent writing data access code does not automatically check whether the caller is allowed to see that data. This skill makes that check mandatory.

## What to Check Before Writing or Modifying Access-Sensitive Code

Before generating or modifying any route, endpoint, middleware, or data query, answer each of the following:

**1. Is there an authorization check?**
Every endpoint or function that accesses user-specific data must verify the caller's identity and permissions before touching any record. "Authenticated" is not the same as "authorized."

**2. Is object-level authorization enforced?**
If the code retrieves a record by ID (e.g., `GET /invoices/4821`), confirm the owner of record 4821 matches the authenticated caller. This is the IDOR pattern. Look for any query that uses a user-supplied ID without a corresponding ownership check.

**3. Are vertical privilege escalations blocked?**
If the function has role-based behavior (admin vs. user), confirm the role check happens server-side. Client-supplied roles, JWT claims that are trusted without verification, and boolean flags like `isAdmin` passed in request bodies are common fail patterns.

**4. Is the default access posture "deny"?**
New routes and functions should fail closed. If no explicit permission check exists, access should be denied, not allowed.

## Output Format

Before writing any access-sensitive code, produce this report:

```
## Access Control Check

**Object type:** [what is being accessed or modified]
**Caller identity source:** [where the authenticated identity comes from — session, JWT, API key]
**Authorization check present:** [yes / no / not yet implemented]
**IDOR risk:** [present / not present / needs verification]

### Findings
[List each pattern found or not found. Be specific: file name and line number if reviewing existing code.]

### Verdict
[CLEAR — safe to write or modify / BLOCKED — describe what must be added before proceeding]
```

## Hard Stops

Stop and require resolution before writing code if any of the following are true:

- A route accesses records by user-supplied ID with no ownership check
- Authorization logic is implemented on the client side only
- A new admin or privileged function has no role verification
- The access control check is commented out, disabled for "testing," or marked TODO

## What Good Looks Like

```python
# Good: ownership verified before access
invoice = db.query(Invoice).filter(
    Invoice.id == invoice_id,
    Invoice.owner_id == current_user.id  # ownership check
).first()
if not invoice:
    raise HTTPException(status_code=404)
```

```python
# Bad: ID accepted without ownership check
invoice = db.query(Invoice).filter(Invoice.id == invoice_id).first()
return invoice  # any authenticated user can read any invoice
```

An agent that writes the second pattern without flagging it is introducing a vulnerability. This skill stops that.
