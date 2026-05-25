---
name: ssrf-check
description: Checks for server-side request forgery risk before any code that makes outbound HTTP requests based on user-supplied URLs or parameters is written or modified.
when_to_use: Apply automatically when writing or modifying code that makes HTTP requests to URLs derived from user input, fetches remote resources, processes webhooks, or accepts callback URLs.
---

# Server-Side Request Forgery (SSRF) Skill

SSRF happens when an application fetches a URL that the server controls but the user supplied. The server makes a request on the user's behalf, and if there is no restriction on where it can go, the user can point it at internal services, cloud metadata endpoints, or localhost.

The classic SSRF target is `http://169.254.169.254/latest/meta-data/` — the AWS metadata endpoint. From a server inside a VPC, that URL returns IAM credentials. A user who can supply the URL gets the credentials.

## What to Check Before Writing Outbound-Request Code

**1. Is the URL user-supplied?**
Any code where the destination URL or hostname comes from user input, a webhook payload, a URL parameter, or a database record that a user could have written is SSRF-exposed.

**2. Is there an allowlist?**
User-supplied URLs must be validated against an explicit allowlist of permitted hostnames or domains before the request is made. A blocklist (blocking 169.254.x.x, localhost, 127.0.0.1) is insufficient — IPv6 equivalents, DNS rebinding, and decimal/octal IP representations bypass blocklists.

**3. Are internal network ranges blocked at the network layer?**
Application-level allowlists must be backed by network-level controls. If a server can reach internal services, an allowlist bypass can still reach them. Confirm whether outbound requests from the application are network-restricted.

**4. Does the response get returned to the user?**
Blind SSRF (no response returned) is lower severity than reflected SSRF (response returned to user). Both require mitigation, but reflected SSRF is the higher priority.

## Output Format

```
## SSRF Check

**Request trigger:** [webhook / URL parameter / user form input / API payload / other]
**URL source:** [fully user-controlled / partially user-controlled (domain fixed, path variable) / internal only]
**Response handling:** [returned to user (reflected) / not returned (blind)]

### Findings
- Allowlist present: [yes — list permitted domains / no — BLOCKED]
- Internal IP ranges blocked: [application-level / network-level / neither — BLOCKED]
- Metadata endpoint (169.254.169.254) blocked: [yes / no — BLOCKED if no and cloud-hosted]
- DNS rebinding mitigated: [yes (resolved IP re-checked after DNS) / no — flag]

### Verdict
[CLEAR / BLOCKED — list each SSRF risk]
```

## Hard Stops

Block before writing outbound request code if:

- The destination URL is user-supplied with no allowlist validation
- The AWS metadata endpoint (169.254.169.254 or fd00:ec2::254) is not blocked for cloud-hosted applications
- Internal network ranges (10.x.x.x, 172.16.x.x, 192.168.x.x, localhost, 127.0.0.1) are reachable from the application server with no application or network control
- The resolved IP address is not re-verified after DNS resolution (DNS rebinding mitigation)

## What Good Looks Like

```python
import ipaddress
import socket
from urllib.parse import urlparse

ALLOWED_HOSTS = {"api.trusted-partner.com", "webhooks.approved-service.io"}

def safe_fetch(url: str) -> requests.Response:
    parsed = urlparse(url)
    if parsed.hostname not in ALLOWED_HOSTS:
        raise ValueError(f"Host {parsed.hostname} not in allowlist")

    # Verify resolved IP is not internal after DNS
    ip = socket.gethostbyname(parsed.hostname)
    addr = ipaddress.ip_address(ip)
    if addr.is_private or addr.is_loopback or addr.is_link_local:
        raise ValueError(f"Resolved IP {ip} is internal — SSRF risk")

    return requests.get(url, timeout=5)
```
