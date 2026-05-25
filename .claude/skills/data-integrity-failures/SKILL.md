---
name: data-integrity-failures
description: Checks deserialization safety, CI/CD pipeline integrity, and dependency integrity before any code that deserializes objects or executes pipeline scripts is written or modified.
when_to_use: Apply automatically when writing or modifying deserialization logic, CI/CD pipelines, update mechanisms, package integrity checks, or any code that processes serialized data from an untrusted source.
---

# Software and Data Integrity Failures Skill

Data integrity failures happen when an application processes serialized data, pipeline steps, or software updates without verifying they have not been tampered with. An agent writing a deserialization handler or a CI pipeline script does not automatically ask whether the data or the script could have been modified by an attacker.

## What to Check Before Writing Integrity-Sensitive Code

**1. Deserialization of untrusted data**
Python's `pickle`, Java's `ObjectInputStream`, PHP's `unserialize`, and Ruby's `Marshal.load` can execute arbitrary code when given crafted input. These must never be used on data from an untrusted source. Use JSON, protobuf, or a schema-validated format instead.

Untrusted sources: user uploads, external APIs, message queues where producers are not under your control, cached data that could be poisoned.

**2. CI/CD pipeline integrity**
Pipeline scripts that `curl | bash`, `pip install` from unverified sources, or clone from branches without pinning are integrity risks. Pipeline steps should use pinned dependency versions, hash verification where possible, and signed artifacts.

**3. Dependency integrity verification**
Package installs should use lockfiles with hash verification. `npm ci` (not `npm install`) in pipelines. `pip install` with `--require-hashes` in production builds.

**4. Deserialized data as code path**
Any flow where deserialized data influences what code executes (dynamic imports, eval, command construction) must treat the deserialized input as untrusted and validate it strictly.

## Output Format

```
## Data Integrity Failures Check

**Code type:** [deserialization / CI pipeline / dependency install / update mechanism]
**Data source:** [user upload / external API / message queue / internal / other]

### Findings
- Deserialization format: [JSON/protobuf — safe / pickle/ObjectInputStream — BLOCKED if untrusted source]
- Pipeline dependency pinning: [pinned with hashes / unpinned — flag]
- Lockfile used in CI: [yes / no — flag]
- Deserialized data used in code execution path: [yes — describe / no]

### Verdict
[CLEAR / BLOCKED — list each integrity risk]
```

## Hard Stops

Block before writing code if:

- `pickle.loads`, `ObjectInputStream`, `unserialize`, or `Marshal.load` is used with data from any untrusted source
- A CI pipeline step uses `curl | bash` or downloads scripts without hash verification
- Deserialized data is passed to `eval`, `exec`, `import`, or a shell command
- Package installs in CI do not use a lockfile

## What Good Looks Like

```python
# Good: JSON deserialization with schema validation
import json
from pydantic import BaseModel

class UserPayload(BaseModel):
    user_id: int
    action: str

data = UserPayload(**json.loads(untrusted_input))  # schema-validated

# Bad: pickle from untrusted source
import pickle
obj = pickle.loads(request.body)  # arbitrary code execution possible
```
