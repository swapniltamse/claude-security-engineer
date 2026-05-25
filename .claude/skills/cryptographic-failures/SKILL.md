---
name: cryptographic-failures
description: Checks for weak or missing encryption before any code handling passwords, tokens, PII, or sensitive data is written or modified.
when_to_use: Apply automatically when writing or modifying code that handles passwords, API keys, tokens, PII, health data, financial data, or any data transmitted over a network.
---

# Cryptographic Failures Skill

Cryptographic failures happen when sensitive data is exposed because it was stored or transmitted without adequate protection. An agent generating data persistence or transmission code defaults to what works, not what is safe. MD5 works. SHA1 works. Storing passwords in plaintext works. None of them are acceptable.

## What to Check Before Writing Crypto-Adjacent Code

**1. Passwords**
Must be stored using an adaptive hashing algorithm: bcrypt, scrypt, Argon2, or PBKDF2. MD5, SHA1, SHA256, and SHA512 are not password hashing algorithms. If the code stores a password, the hash function must be one of the four above.

**2. Sensitive data at rest**
PII, financial data, health records, and API keys must be encrypted at rest. "Stored in the database" is not the same as "encrypted." Check whether the column, file, or object is encrypted. If the ORM or storage layer does not handle it, application-level encryption is required.

**3. Sensitive data in transit**
All transmission of sensitive data must use TLS 1.2 or higher. HTTP endpoints that accept credentials, tokens, or PII are a block condition.

**4. Weak algorithms**
Flag and block use of: MD5, SHA1, DES, 3DES, RC4, ECB mode for block ciphers. These are broken or deprecated for security use.

**5. Hardcoded secrets**
API keys, passwords, encryption keys, or tokens hardcoded in source code are a block condition. Secrets belong in environment variables or a secrets manager, not in code.

## Output Format

```
## Cryptographic Failures Check

**Data type handled:** [passwords / PII / tokens / financial / other]
**Storage method:** [database column / file / object store / in-memory only]
**Transmission method:** [HTTPS / HTTP / internal only]

### Findings
- Password hashing algorithm: [algorithm name or "not applicable"]
- Encryption at rest: [yes / no / not applicable]
- Transport security: [TLS version or "HTTP — BLOCKED"]
- Weak algorithms detected: [list or "none"]
- Hardcoded secrets detected: [yes / no]

### Verdict
[CLEAR / BLOCKED — list each block condition]
```

## Hard Stops

Block and require resolution before writing code if:

- Password storage uses MD5, SHA1, SHA256, SHA512, or plain text
- Sensitive data is transmitted over HTTP
- A hardcoded secret, key, or credential appears in the code
- ECB mode is used for block cipher encryption
- A deprecated algorithm (DES, 3DES, RC4) is used for any security purpose

## What Good Looks Like

```python
# Good: Argon2 for password hashing
from argon2 import PasswordHasher
ph = PasswordHasher()
hashed = ph.hash(user_password)

# Good: secrets from environment
import os
api_key = os.environ["THIRD_PARTY_API_KEY"]
```

```python
# Bad: SHA256 for password storage
import hashlib
hashed = hashlib.sha256(password.encode()).hexdigest()

# Bad: hardcoded secret
API_KEY = "sk-live-abc123def456"
```
