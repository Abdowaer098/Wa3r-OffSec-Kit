# IDOR (Insecure Direct Object Reference)

## Overview
IDOR occurs when an application uses user-supplied input to directly access objects (database records, files, resources) without verifying that the user is authorized to access that specific object. It is the most common access control vulnerability and frequently leads to unauthorized data access, modification, or deletion.

## When This Happens

- URLs or API parameters contain sequential IDs: `/api/users/123`, `/invoice/456`
- UUIDs in requests that reference other users' resources
- Any endpoint where changing a parameter value returns a different user's data
- File download/view features: `/download?file=report_123.pdf`

## Recon / Detection

### 1. Create two accounts
Always test IDOR with two accounts (Account A and Account B):
- Log in as Account A → capture requests that reference Account A's resources
- Replace A's identifiers with B's → does B's data leak?

### 2. Identify direct object references

| Pattern | Location |
|---------|----------|
| `/api/users/{id}` | Path parameter |
| `?user_id=123` | Query parameter |
| `{"order_id": 456}` | JSON body |
| `X-User-Id: 789` | Custom header |
| `/files/user_123/report.pdf` | Path-based file access |

### 3. Test ID patterns

| ID type | Test approach |
|---------|--------------|
| Sequential integers | Decrement/increment: 123 → 122, 124 |
| UUIDs | Swap between accounts |
| Email as ID | Replace with victim's email |
| Hashed/encoded IDs | Decode (base64, MD5) → modify → re-encode |
| Composite keys | Modify individual components |

## Exploitation Steps

### 1. Horizontal privilege escalation (same role, different user)
```
GET /api/profile/123   → Your profile
GET /api/profile/124   → Another user's profile? → IDOR!
```

### 2. Vertical privilege escalation (access higher role)
```
GET /api/admin/dashboard   → 403 Forbidden
GET /api/users/1           → returns admin user data? → IDOR + info disclosure
PUT /api/users/1 {"role": "admin"}   → Role change? → IDOR + privilege escalation
```

### 3. IDOR on write operations (more impactful)
```
PUT /api/users/VICTIM_ID {"email": "attacker@evil.com"}   → Account takeover
DELETE /api/orders/VICTIM_ORDER_ID                         → Data deletion
POST /api/transfer {"to": "attacker", "from": "VICTIM_ID", "amount": 1000}  → Financial
```

### 4. IDOR on file access
```
GET /download?file=invoice_123.pdf   → Your invoice
GET /download?file=invoice_124.pdf   → Another user's invoice?
GET /download?file=../../etc/passwd  → Path traversal?
```

### 5. IDOR via parameter pollution
```
GET /api/profile?id=123&id=456         → Which ID is used?
GET /api/profile?id=123,456            → Array injection?
GET /api/profile?id[]=123&id[]=456     → Array parameter?
```

### 6. UUID IDOR techniques
UUIDs are harder to guess but can be leaked via:
- API responses that include other users' UUIDs
- User listings, search results, comments
- Referrer headers
- Previous API calls (e.g., user creation returns UUID)
- UUID v1 contains timestamp — can be predicted (see password-reset-abuse.md)

## Testing Checklist

| HTTP Method | Test | Impact |
|-------------|------|--------|
| **GET** | Read another user's data | Information disclosure |
| **PUT/PATCH** | Modify another user's data | Data tampering |
| **DELETE** | Delete another user's resources | Data destruction |
| **POST** | Create resources as another user | Impersonation |

Always test IDOR on **every CRUD operation**, not just GET.

## Tools

- **Burp Suite**: Repeater for manual testing, Autorize extension for automated auth testing.
- **Autorize** (Burp extension): Automatically replays requests with different session tokens — highlights IDOR.
- **OWASP ZAP**: Access control testing.
- **Custom scripts**: Iterate through IDs and compare responses.

## Real-World Notes

- IDOR is the **#1 most common vulnerability** in bug bounty programs (HackerOne data).
- **Write IDORs** (PUT/DELETE/POST) are much higher severity than read IDORs — always test both.
- Don't stop at `/api/users/{id}` — test every endpoint that references an object: orders, messages, files, settings, payments.
- **Autorize** Burp extension is essential — it automatically detects IDOR by replaying every request with a different user's session.
- Check for IDOR in **WebSocket messages** and **GraphQL queries** too, not just REST APIs.

## References

- `techniques/mass-assignment-vulnerabilities.md`
- `techniques/password-reset-abuse.md`
- `methodology/exploitation-methodology.md`
