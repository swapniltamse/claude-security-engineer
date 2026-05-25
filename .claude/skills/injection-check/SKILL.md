---
name: injection-check
description: Scans all code that constructs queries, commands, or markup from user input, blocking SQL injection, command injection, LDAP injection, and XSS before the code ships.
when_to_use: Apply automatically when writing or modifying database queries, shell commands, LDAP queries, XML parsers, HTML rendering, or any code that incorporates user-supplied input into a structured language.
---

# Injection Check Skill

Injection is the class of vulnerability where user-supplied input is interpreted as part of a command or query rather than as data. An agent generating database or shell code will often use string formatting because it is simpler. Simpler is what gets exploited.

## What to Check Before Writing Input-Handling Code

**SQL Injection**
Scan for string concatenation or f-string/format-string construction of SQL queries. Every parameterized query is safe. Every concatenated query is a vulnerability.

Patterns to flag:
- `"SELECT * FROM users WHERE id = " + user_id`
- `f"SELECT * FROM users WHERE name = '{name}'"`
- `cursor.execute("DELETE FROM orders WHERE id = %s" % order_id)` (note: `%s` with `%` substitution, not parameterized)
- Raw query builders that accept unsanitized field names for ORDER BY or column selection

**Command Injection**
Any code that passes user input to `os.system`, `subprocess`, `exec`, `eval`, shell=True, or equivalent in any language is a block condition unless the input is fully validated against a whitelist.

Patterns to flag:
- `os.system(f"convert {filename} output.pdf")`
- `subprocess.run(user_command, shell=True)`
- `eval(user_expression)`

**XSS (Cross-Site Scripting)**
HTML rendered to a browser that includes unsanitized user input is an XSS vulnerability. Flag any template or string that places user-supplied values directly into HTML without escaping. Server-side rendering frameworks that auto-escape are acceptable only if auto-escape is confirmed to be on.

**LDAP Injection**
LDAP queries constructed from user input without escaping are injectable. Flag any LDAP filter construction that uses string concatenation with user input.

## Output Format

```
## Injection Check

**Input sources identified:** [list of places user input enters the code]
**Query/command types constructed:** [SQL / shell / HTML / LDAP / XML / other]

### Findings
[For each finding: pattern detected, file and line, severity]

### Verdict
[CLEAR / BLOCKED — list each injection risk found]
```

## Hard Stops

Block before writing code if:

- A SQL query is constructed via string concatenation or formatting with user input
- User input is passed to a shell command without whitelist validation
- HTML output includes user input without confirmed escaping
- `eval()` or `exec()` is called with any user-influenced value
- `shell=True` is used with subprocess and the command includes user input

## What Good Looks Like

```python
# Good: parameterized SQL
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# Good: subprocess without shell, arguments as list
result = subprocess.run(["convert", filename, "output.pdf"], capture_output=True)

# Good: ORM with parameterization built in
user = User.objects.filter(id=user_id).first()
```

```python
# Bad: SQL concatenation
cursor.execute("SELECT * FROM users WHERE id = " + user_id)

# Bad: shell command with user input
os.system(f"convert {request.args['file']} output.pdf")
```
