---
name: security-misconfiguration
description: Checks for insecure default settings, exposed debug endpoints, permissive CORS, open cloud storage, and missing security headers before any configuration or deployment code ships.
when_to_use: Apply automatically when writing or modifying server configuration, cloud infrastructure code, CORS settings, HTTP response headers, environment setup, or deployment scripts.
---

# Security Misconfiguration Skill

Security misconfiguration is the gap between a framework's default settings (permissive, developer-friendly) and what a production system should allow (restrictive, hardened). An agent generating server config or cloud resources starts from defaults. Defaults are not production-ready.

## What to Check Before Writing Configuration Code

**1. Debug mode and error verbosity**
Debug mode must be off in production. Detailed error messages (stack traces, SQL queries, environment variables) must not be returned to clients. Check: `DEBUG=True`, `ENV=development`, verbose error handlers, framework-level debug flags.

**2. Default credentials**
Default usernames and passwords on databases, admin panels, and services must be changed before deployment. Flag any infrastructure code that deploys a service without explicitly setting credentials.

**3. CORS policy**
`Access-Control-Allow-Origin: *` is a block condition for any endpoint that handles authenticated requests or sensitive data. CORS must specify exact allowed origins.

**4. Security headers**
HTTP responses for browser-facing applications must include:
- `Content-Security-Policy`
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY` or `SAMEORIGIN`
- `Strict-Transport-Security` (for HTTPS endpoints)
- `Referrer-Policy`

**5. Cloud storage permissions**
S3 buckets, GCS buckets, and Azure Blob containers must not be publicly readable unless the explicit purpose is public content delivery. Flag any `public-read` ACL or equivalent.

**6. Open ports and unnecessary services**
Infrastructure code that exposes ports beyond what the application requires is a block condition. Database ports must not be exposed to the public internet.

## Output Format

```
## Security Misconfiguration Check

**Configuration type:** [server / cloud infra / CORS / headers / credentials / other]

### Findings
- Debug mode: [off / on — BLOCKED / not applicable]
- Default credentials: [changed / not changed — BLOCKED / not applicable]
- CORS policy: [specific origins / wildcard — BLOCKED if wildcard on authenticated endpoint]
- Security headers: [present / missing — list which ones]
- Cloud storage ACL: [private / public — BLOCKED if public with no documented reason]
- Exposed ports: [list open ports and whether each is required]

### Verdict
[CLEAR / BLOCKED — list each misconfiguration]
```

## Hard Stops

Block before shipping config if:

- `DEBUG=True` or equivalent is present without an environment variable guard
- CORS allows all origins (`*`) on an authenticated endpoint
- A cloud storage bucket is set to public-read with no documented justification
- A database port is exposed to 0.0.0.0 or the public internet
- Required security headers are absent from browser-facing responses
- Default credentials are not changed before deployment

## What Good Looks Like

```python
# Good: debug mode from environment
DEBUG = os.environ.get("DEBUG", "false").lower() == "true"

# Good: specific CORS origins
CORS_ALLOWED_ORIGINS = ["https://app.yourcompany.com"]
```

```yaml
# Bad: public S3 bucket
- Bucket: my-app-uploads
  AccessControl: public-read

# Good: private bucket, serve via signed URLs
- Bucket: my-app-uploads
  AccessControl: private
```
