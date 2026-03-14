# Case Study: SSTI to Database Access in Flask/Jinja2 CTF

## Overview
This case study documents the exploitation of a Server-Side Template Injection (SSTI) vulnerability in a Flask/Jinja2 web application during a CTF challenge. The key insight was bypassing keyword filters by accessing the database through an exposed `User` object rather than using blocked system commands.

## Target Context

- **Platform**: Flask web application with Jinja2 templating.
- **Feature**: User profile with a "bio" field that gets rendered through `render_template_string()`.
- **Filters**: Keywords `os`, `class`, `import`, `subclass` were blocked.

## Vulnerability

**Type**: Server-Side Template Injection (SSTI) leading to database access and admin account takeover.

### The vulnerable code pattern

```python
temp = open("shareprofile.html").read()
return render_template_string(temp % bio, User=User)
```

Two critical mistakes:
1. **String formatting before template rendering**: `temp % bio` inserts user input into the HTML template **before** Jinja2 processes it. If the bio contains `{{ }}`, Jinja2 will execute it as code.
2. **Exposing the User model**: `User=User` passes the database ORM model directly to the template context, giving template expressions full query access.

### Why this is dangerous
Normal template rendering is safe:
```python
return render_template("profile.html", username=username)
# Jinja2 auto-escapes {{ username }} — no code execution
```

But with `render_template_string(temp % bio)`:
- User input is placed into the template string **before** Jinja2 sees it.
- Any `{{ }}` in the bio becomes executable Jinja2 code.
- The server is literally saying: "Write code and I'll execute it."

## Exploitation

### 1. Confirming SSTI
Set bio to:
```
{{ 7 * 7 }}
```
If the profile page displays `49` instead of the literal text, SSTI is confirmed.

### 2. Exploring available objects
The `User` object was passed to the template context, so:
```jinja2
{{ User }}
→ Shows the User class/model
```

### 3. Bypassing filters via the User object
The standard SSTI-to-RCE path uses `__class__`, `__subclasses__`, `os`, and `import` — all of which were blocked. But the `User` object was directly connected to the database ORM.

**Extracting the admin's email:**
```jinja2
{{ User.query.filter_by(username='admin').first().email }}
```

This payload:
- Accesses the database through the ORM (`User.query`)
- Filters for the admin user
- Extracts their email address
- **No blocked keywords used** — no `os`, `class`, `import`, or `subclass`

### 4. Escalation path

| Stage | Action | Result |
|-------|--------|--------|
| Information | `{{ User.query.all() }}` | List all users |
| Targeted data | `{{ User.query.filter_by(username='admin').first().email }}` | Admin email |
| Account takeover | Use admin email for password reset | Admin access |
| If RCE needed | Access other ORM models or config | Full compromise |

## Key Insight: "Enter through the back door"

The developer blocked the obvious attack path:
```
os → blocked
class → blocked
import → blocked
subclass → blocked
```

But left a legitimate application object (`User`) exposed in the template context that was **directly connected to the database**. The attacker did not need any of the blocked keywords — they accessed sensitive data through the application's own data model.

> You don't always need to break in through the front door. If the application hands you a key to the back door (an ORM model in the template context), use it.

## Lessons Learned

1. **Keyword-based filtering is insufficient** — attackers will find alternative paths to reach their goal.
2. **Never pass ORM models to template contexts** — expose only the specific, safe data fields needed.
3. **`render_template_string(user_input)` is almost always vulnerable** — use `render_template()` with auto-escaping instead.
4. **String formatting before template rendering** (`temp % bio`) defeats all of Jinja2's built-in safety mechanisms.
5. **SSTI severity is often underestimated** — it frequently leads from information disclosure to full RCE.
6. **The threat progression** is: information leak → credential theft → account takeover → system compromise.

## References

- `techniques/server-side-template-injection.md`
- `methodology/web-recon-methodology.md`
