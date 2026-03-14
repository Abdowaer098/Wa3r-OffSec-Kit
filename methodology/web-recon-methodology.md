# Web Recon Methodology

## Overview
A structured, repeatable workflow for web application reconnaissance, combining passive discovery, active enumeration, JavaScript analysis, and historical data mining. This methodology is distilled from multiple real bug bounty programs and internal notes.

## When This Happens
Use this playbook whenever you start work on a new target (program, customer, or lab) and need to:

- Understand the attack surface.
- Discover subdomains, technologies, and exposed services.
- Map application routes, APIs, and parameters.
- Prepare for deeper vulnerability testing.

## Recon / Detection

### 1. Baseline fingerprinting
- Identify what you are attacking and how it responds.
- Typical commands:

```bash
whatweb https://target.com
nmap -sV -p 80,443 https://target.com --script http-enum
```

Focus on:
- HTTP status patterns
- Server/technology stack
- Obvious admin panels or secondary apps

### 2. Content and directory discovery
Use multiple wordlists and tools to map reachable paths:

```bash
ffuf -w /usr/share/wordlists/dirb/big.txt -u https://target.com/FUZZ -o dirs_main.json
gobuster dir -u https://target.com -w /usr/share/wordlists/dirb/big.txt -o gobuster_main.txt
```

Targets:
- `/admin`, `/management`, `/internal`, `/config`, `/api`, `/dev`
- Technology-specific panels (`/wp-admin`, `/phpmyadmin`, etc.)

### 3. CMS and framework specific recon
If the target runs a known CMS or framework, run focused enumeration:

```bash
wpscan --url https://target.com --enumerate p,t,tt,u
droopescan scan drupal -u https://target.com
joomscan -u https://target.com
```

Output:
- Plugin/theme versions
- Users and routes
- Known vulnerable components

### 4. API and microservice discovery
Bruteforce and pattern-based discovery for API routes:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
     -u https://target.com/FUZZ -o api_dirs.json

ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-seen-in-wild.txt \
     -u https://target.com/api/FUZZ -o api_endpoints.json
```

Look for:
- `/api`, `/v1`, `/v2`, `/graphql`, `/rest`, `/internal`
- Open or weakly protected management endpoints

### 5. Subdomain and DNS reconnaissance
Enumerate subdomains and enrich them with DNS and HTTP probing:

```bash
subfinder -d target.com -o subs_raw.txt
httpx -l subs_raw.txt -status-code -title -tech-detect -o live_subs.txt
```

(If you use a wrapper tool, plug these in there.)

Follow up with:
- Virtual host and DNS recon tools for additional edges.
- Simple manual review of `live_subs.txt` to pick promising hosts.

### 6. Screenshots and visual triage
Generate screenshots and quickly visually inspect the surface:

```bash
gowitness file -f live_subs.txt
aquatone -ports xlarge < live_subs.txt
```

Goal:
- Quickly spot login panels, admin consoles, debug views, and “forgotten UIs”.

### 7. Parameter and workflow discovery
For interesting pages and APIs, enumerate parameters:

```bash
arjun -u https://target.com/login -o login_params.txt
arjun -u https://target.com/search -m POST -o search_params.txt
arjun -u https://target.com/api/user -o api_params.txt
```

Use this to:
- Build fuzzing payloads.
- Guide logic and mass-assignment testing.

### 8. JavaScript and client-side recon
Combine automated JS crawlers and targeted analysis:

See `methodology/javascript-endpoint-discovery.md` and `tools/recon-tools.md` for details, but at a high level:
- Collect JS URLs (wayback, gau, crawling).
- Extract hidden endpoints, parameters, tokens, and third-party integrations.
- Feed new routes back into dir fuzzing and parameter discovery.

### 9. Historical and archive recon
Use historical data to find legacy endpoints and parameters:

```bash
waybackurls target.com > wayback_urls.txt
gau target.com > gau_urls.txt
httpx -l wayback_urls.txt -o accessible_old.txt
```

Targets:
- Old admin panels.
- Deprecated APIs still reachable.
- Unpatched legacy paths.

### 10. API schemas and service introspection
Probe for introspection and documentation:

```bash
curl -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}' > graphql_schema.txt

ffuf -w swagger_paths.txt -u https://target.com/FUZZ -o swagger_scan.json
ffuf -w soap_paths.txt -u https://target.com/FUZZ -o soap_scan.json
```

If you find:
- Open GraphQL schemas
- Swagger/OpenAPI specs
- WSDL / SOAP interfaces

…use them to:
- Enumerate operations, types, and inputs.
- Identify high-risk business operations for deeper testing.

### 11. OSINT, GitHub, and leak hunting
Search public code and paste sites for references to the target:

```bash
gh search code "target.com" > github_repos.txt
gh search code "api_key target.com" > github_keys.txt

curl -s "https://psbdmp.ws/api/search/target.com" > pastebin_results.txt
```

Look for:
- API keys and secrets.
- Internal hostnames and endpoints.
- Example payloads that reveal business logic.

## Exploitation Steps (High-Level)
This playbook is recon-focused. Once you have:

- A map of hosts and technologies.
- A list of routes and APIs.
- Parameters and input shapes.
- JS-derived hidden functionality.

You pivot into specific techniques:

- **Mass assignment / logic flaws** on JSON-based APIs.
- **SSTI/XXE/RCE** using parameters discovered by tools like Arjun and JS analysis.
- **Broken access control** on admin or “internal” routes found through subdomain/dir brute forcing.

Each of these exploit paths is documented in depth under `techniques/` and `scenarios/`.

## Example Payloads
This methodology stage focuses on discovery, so payloads are usually:

- Generic path fuzzing: `FUZZ`, `*`, `{id}`, `/api/v1/FUZZ`
- Basic parameter fuzzing: `test`, `123`, `'`, `"`, `true`, `false`
- Technology-specific checks: `<?xml`, `{{7*7}}`, `${{7*7}}`, `<script>alert(1)</script>`

Concrete payloads live in:

- `tools/wordlists-and-payload-lists.md`
- Technique-specific docs like `techniques/server-side-template-injection.md`

## Tools

- **Surface & tech detection**: `whatweb`, `nmap`, `httpx`
- **Dir and API discovery**: `ffuf`, `gobuster`, curated wordlists
- **CMS recon**: `wpscan`, `droopescan`, `joomscan`
- **Subdomain & DNS**: `subfinder`, `dnsx` (or equivalents)
- **Screenshots**: `gowitness`, `aquatone`
- **Parameter discovery**: `arjun`
- **JS & content recon**: your JS endpoint tools and `WaerJScan`
- **Archive recon**: `waybackurls`, `gau`
- **OSINT**: `gh` (GitHub CLI), paste search APIs

See `tools/recon-tools.md` for a more exhaustive, command-focused catalog.

## Real-World Notes

- Many of your successful bugs start from **simple recon done very deeply** on a small subset of hosts rather than shallow scans of everything.
- JS recon and archive recon regularly reveal “hidden” administrative or testing functionality that is not obvious through basic dir fuzzing.
- Consistent naming for output files (e.g., `*_dirs.json`, `*_params.txt`, `*_urls.txt`) helps you re-use recon artifacts across targets and automate follow-up.

## References

- `tools/recon-tools.md`
- `methodology/javascript-endpoint-discovery.md`
- `techniques/mass-assignment-vulnerabilities.md`
- `techniques/server-side-template-injection.md`
