# JavaScript Endpoint Discovery

## Overview
JavaScript files often reveal hidden endpoints, parameters, and client-side logic that are not visible from simple crawling or directory brute forcing. This workflow combines bookmarklets, custom scanners, and automation to turn JS into a high-signal recon source.

## When This Happens

Use this methodology when:

- You have identified interesting web apps and want deeper coverage beyond `/robots.txt` and directory fuzzing.
- The application is single-page or API-heavy (React, Vue, Angular, etc.).
- You suspect hidden admin panels, debugging endpoints, or feature flags only referenced in JS.

## Recon / Detection

### 1. Quick in-browser JS scraping (bookmarklet)

From your `Javascript scan.md` note:

```javascript
javascript:(async function(){ const regex = /['"`]\/([a-zA-Z0-9_\/\-\.\+]+)['"`]/g; const results = new Set(); const scripts = document.getElementsByTagName("script"); async function processScript(script) { try { let content = ""; if (script.src) { const response = await fetch(script.src); content = await response.text(); } else { content = script.textContent; } let match; while ((match = regex.exec(content)) !== null) { results.add(match[1]); } } catch (e) { console.error("Error processing script:", e); } } // Process all scripts await Promise.all(Array.from(scripts).map(processScript)); // Process page content once const pageContent = document.documentElement.outerHTML; let match; while ((match = regex.exec(pageContent)) !== null) { results.add(match[1]); } // Display results document.body.innerHTML = Array.from(results).join("<br>"); })();
```

Usage:
- Save this as a browser bookmark.
- On an interesting page, click it to extract path-like strings from embedded and external scripts.
- Copy out likely endpoints such as `/api/user`, `/v1/orders`, `/admin/dashboard`, etc.

### 2. Advanced JS crawling with WaerJScan

From `WaerJScan.md`, you have a full-featured crawler that:

- Crawls HTML and JS for:
  - Endpoints and paths
  - Parameters
  - API keys and secrets
  - Forms and technologies
- Generates a custom wordlist using TF-IDF and n-grams.

Typical usage:

```bash
python3 waerjscan.py -u https://target.com -d 3 -o waerjscan_results --delay 0.1 --threads 15
```

Key outputs to feed back into recon:
- `endpoints.txt` – URLs and paths discovered from JS.
- `parameters.txt` – Names of parameters to fuzz.
- `custom_wordlist.txt` – High-value tokens for dir and parameter brute forcing.

### 3. Historical JS via archives

Combine JS scanners with archive tools:

```bash
waybackurls target.com | grep "\.js" | sort -u > js_wayback.txt
gau target.com | grep "\.js" | sort -u > js_gau.txt
cat js_wayback.txt js_gau.txt | sort -u > js_all.txt
```

Then:
- Crawl or download these JS files.
- Run WaerJScan or your own regex-based analysis.
- Look for older, deprecated endpoints that may still be reachable.

### 4. Prioritization

Not all JS endpoints are equal. Prioritize:

- Paths containing: `admin`, `internal`, `debug`, `beta`, `dev`, `test`, `v1`, `v2`.
- Actions like: `/login`, `/reset-password`, `/order`, `/payment`, `/config`, `/feature-flags`.
- API keys and base URLs pointing to different backends (e.g., auth, payments, analytics).

## Exploitation Steps

Once you have extracted endpoints:

1. **Validate reachability**
   - Use `httpx` or `curl` to check which endpoints return meaningful responses.

2. **Map methods and auth**
   - Try `GET`, `POST`, `PUT`, etc.
   - Check whether the endpoint is protected by authentication or misconfigured CORS.

3. **Probe for vulnerabilities**
   - Use parameters discovered from JS (`parameters.txt`) with:
     - Mass assignment testing.
     - SSTI, XXE, injection payloads where appropriate.
     - Business-logic abuse based on how JS expects flows to work.

4. **Feed into automation**
   - Add discovered endpoints and parameters to:
     - Burp Suite (for manual and automated testing).
     - Fuzzers/wordlists for `ffuf` or custom tools.

## Example Payloads

Examples are dependent on the vulnerability class, but common ones include:

- Detection:
  - Simple benign values: `test`, `123`, `true`, `false`.
- Template/injection checks:
  - `{{7*7}}`, `${{7*7}}`, `<?xml version="1.0"?><!DOCTYPE xxe [...]>`.
- Mass assignment / logic:
  - Extra fields like `"role": "admin"`, `"is_admin": true`, `"item_price": 0`, `"discount": 100`.

For full payload banks, see:
- `tools/wordlists-and-payload-lists.md`
- Technique docs under `techniques/`.

## Tools

- **In-browser bookmarklet** – Lightweight and fast for single pages.
- **WaerJScan** – Your primary JS-centric recon and analysis tool.
- **Archive tools** – `waybackurls`, `gau` for historical JS.
- **Support tools** – `httpx`, `curl`, `Burp Suite`, `gf` and others to follow up on endpoints.

## Real-World Notes

- JS frequently exposes “internal” APIs that are not linked in HTML navigation.
- Historical JS is gold: old endpoints often preserve dangerous behaviors even after they’ve disappeared from the UI.
- Automatically generated wordlists from WaerJScan align directly with the target’s own vocabulary, which is usually more effective than generic wordlists.

## References

- `tools/recon-tools.md`
- `tools/wordlists-and-payload-lists.md`
- `methodology/web-recon-methodology.md`
