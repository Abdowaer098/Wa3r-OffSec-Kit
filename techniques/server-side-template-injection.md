# Server-Side Template Injection (SSTI)

## Overview
Server-Side Template Injection (SSTI) occurs when user-controlled input is passed into a server-side template engine and evaluated as template code instead of being treated as plain data. In Python (e.g., Flask/Jinja2), this often allows an attacker to execute expressions like `{{ 7*7 }}`, access sensitive objects, pull data from the database, and in some cases escalate to remote code execution (RCE).

## When This Happens

You should suspect SSTI when:

- The application uses a template engine (Jinja2, Twig, Freemarker, etc.).
- User input appears reflected in a rendered template, especially inside `{{ ... }}` or `{% ... %}` blocks.
- The backend uses functions like `render_template_string` or builds templates dynamically with user input.
- Changing your input from a string to an expression (e.g., `{{ 7*7 }}`) changes the rendered output.

In the Flask/Jinja2-style pattern from your notes, a dangerous construct is:

```python
temp = open("shareprofile.html").read()
return render_template_string(temp % bio, User=User)
```

Here:
- `bio` is user-controlled.
- `temp % bio` injects `bio` into the template **before** Jinja renders it.
- `User=User` exposes the `User` object directly to the template, enabling DB access.

## Recon / Detection

### 1. Find template sinks

Look for:

- Template functions: `render_template`, `render_template_string`, `env.from_string`, etc.
- Views that render user-controlled text such as profile bios, comments, or “share” pages.
- Error messages that reference template engines.

### 2. Probe with harmless expressions

Start from simple payloads:

- `test` → confirms reflection.
- `{{ 7*7 }}` → if output changes to `49`, code is being evaluated.
- `{{ 7*'7' }}` → may print `7777777`, another sign of evaluation.

If you see evaluation instead of literal output, you likely have SSTI.

### 3. Enumerate accessible objects

Especially when the backend passes named objects (like `User`) into the template context, try:

```jinja
{{ User }}
{{ User.query }}
{{ User.query.first() }}
```

Observe:
- Types or string representations in the response.
- Whether you can chain attributes and methods (e.g., `query.filter_by(...).first().email`).

## Exploitation Steps

### Example scenario: Database data exfiltration

In your CTF-style note, the backend explicitly exposed `User` in the template:

```python
return render_template_string(
    temp % bio,
    User=User
)
```

Payload progression:

1. Confirm SSTI:
   ```jinja
   {{ 7*7 }}
   ```
   If response shows `49`, the template engine is executing your expression.

2. Access the `User` object:
   ```jinja
   {{ User.query.filter_by(username='admin').first().email }}
   ```
   If successful, this pulls the admin’s email directly from the database and prints it in the template.

3. Extend to more sensitive data:
   - Enumerate other fields: `id`, `password_hash`, `is_admin`, etc.
   - Combine with other app logic to escalate privileges or pivot.

In many real applications, similar techniques allow:
- Reading sensitive records.
- Bypassing authorization checks by directly querying ORM models.

### Example scenario: Beyond DB access (where possible)

Depending on the template engine and configuration, SSTI can sometimes be extended to:

- Access dangerous attributes (e.g., Python’s `__mro__`, `__subclasses__`).
- Import modules like `os` and call functions.

However, your note also highlights a more subtle point:
- Even when obvious keywords (`os`, `import`, `class`, `subclass`) are blocked, **exposed objects** like `User` can still be abused to reach sensitive data without classic RCE.

## Example Payloads

- Detection:
  ```jinja
  {{ 7*7 }}
  ```

- Type/structure inspection:
  ```jinja
  {{ User }}
  {{ User.__class__ }}
  ```

- ORM-based data extraction:
  ```jinja
  {{ User.query.filter_by(username='admin').first().email }}
  ```

Adjust object names (`User`, `Post`, `Account`, etc.) based on what the backend passes into the template context or what you infer from errors and code.

## Tools

- **Burp Suite**: To inject and iterate over payloads in parameters and body fields.
- **Browser dev tools**: To locate where user input is rendered in the DOM.
- **Wordlists / snippets**: Small SSTI probes for different engines (Jinja2, Twig, Freemarker).
- **Source review** (when available): To identify template rendering functions and objects passed into the context.

## Real-World Notes

- In practice, SSTI often starts as a “data leak” bug (e.g., exposing admin email), but the same primitive can escalate into full control over database data or even RCE.
- Many defenses focus on blocking obvious strings (`os`, `import`), but the real fix is:
  - Avoid building templates from untrusted data (`temp % bio`).
  - Avoid passing powerful objects (like ORM models) directly into templates.
- Your CTF note is a good example of “smart” exploitation:
  - Instead of fighting filters, you used the exposed `User` object to pivot into the database cleanly.

## References

- `methodology/web-recon-methodology.md`
- `methodology/javascript-endpoint-discovery.md`
- `tools/recon-tools.md`

