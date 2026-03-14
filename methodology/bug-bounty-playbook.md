# Bug Bounty Playbook

## Overview
A high-level playbook for approaching bug bounty programs, from program selection through reporting. This document consolidates lessons from practical experience and structured training, covering mindset, workflow, and tactics for finding impactful vulnerabilities.

## Program Selection

### Choosing your target
| Factor | What to look for |
|--------|-----------------|
| **Scope size** | Larger scope = more attack surface. Wildcard domains (`*.target.com`) are ideal |
| **Bounty range** | Match your effort to the payout. High bounties attract competition but reward depth |
| **Program maturity** | Newer programs often have more low-hanging fruit |
| **Technology stack** | Pick targets using tech you understand (WordPress, React, Node, etc.) |
| **Response time** | Check program stats — fast triage = faster payouts |

### For large targets (Facebook-scale)
1. **Surface expansion** — Don't scan `target.com`. Scan `*.target.com`.
2. **Historical recon** — Dead parameters live in Wayback Machine archives.
3. **JS source analysis** — Source maps reveal internal API routes.
4. **DOM sink tracing** — Follow data from URL → innerHTML without WAF seeing it.
5. **OAuth flows** — `redirect_uri` and `state` parameters are prime XSS sources.
6. **Error pages** — 404/403/500 pages often reflect raw input with no filtering.
7. **Third-party JS** — Find CSP-whitelisted CDNs with JSONP endpoints (gadgets).
8. **Parser differentials** — mXSS: browser parses differently than server sanitizes.
9. **CORS chains** — CORS misconfig + XSS = account takeover.
10. **Chaining** — Open redirect → javascript: URI → full XSS.

## The Recon-to-Exploit Workflow

### Phase 1: Understand the target
```
whatweb target.com
nmap -sV -p 80,443 target.com --script http-enum
```
Browse manually with Burp proxy. Test all user flows: login, register, cart, payment, profile, settings.

### Phase 2: Map the attack surface
Follow the full methodology in `methodology/web-recon-methodology.md`:
1. Subdomain enumeration
2. Directory and API discovery
3. CMS-specific scanning
4. JavaScript analysis
5. Parameter discovery
6. Historical/archive recon

### Phase 3: Identify vulnerability classes
Based on the technology and features found:

| Feature found | Test for |
|--------------|----------|
| JSON APIs with object binding | Mass assignment, IDOR |
| Template rendering | SSTI |
| File upload (XML, SVG, DOCX) | XXE |
| Password reset flow | Token leakage, host header injection, IDOR |
| JWT authentication | Algorithm confusion, key injection |
| Reflected input | XSS (reflected, DOM) |
| Cross-origin APIs | CORS misconfiguration |
| CDN/caching layer | Web cache poisoning |
| User-controlled URLs | SSRF, open redirect |

### Phase 4: Exploit and document
- Use technique-specific docs from `techniques/` for detailed exploitation steps.
- Always escalate: don't stop at `alert(1)` — show cookie theft, account takeover, or data exfiltration.
- Document with: full request/response, screenshots, impact statement.

## Core Vulnerability Areas (study path)

Based on the CPTE (Certified Penetration Testing Expert) curriculum and real-world experience:

1. **Access Control** — IDOR, privilege escalation, admin panel discovery, role manipulation
2. **Business Logic** — Client-side trust, negative quantity attacks, workflow bypass, state machine manipulation
3. **Injection** — Command injection, SSTI, XXE, SQL injection
4. **Authentication** — Broken auth, JWT attacks, brute force, credential stuffing
5. **SSRF** — Backend SSRF, blind SSRF, blacklist bypass
6. **Client-Side** — XSS (reflected, stored, DOM), prototype pollution, CORS
7. **Recon Automation** — Nuclei, custom tools, Python scripting

## Reporting Tips

### Structure
1. **Title**: Clear, descriptive (`SSRF via XML-RPC Pingback on *.target.com`)
2. **Severity**: Use CVSS or program's scale; justify your rating
3. **Description**: What the vulnerability is and why it matters
4. **Steps to Reproduce**: Exact, numbered steps anyone can follow
5. **Impact**: What an attacker can achieve (data theft, RCE, account takeover)
6. **Proof of Concept**: curl commands, screenshots, video if helpful
7. **Remediation**: Suggest a fix

### Common mistakes
- Submitting `alert(1)` XSS without impact escalation → lower bounty.
- Missing edge cases in reproduction steps → marked as "cannot reproduce."
- Not showing real-world impact → downgraded severity.
- Reporting known/public CVEs without proving exploitability → duplicate or N/A.

## Time Management

- **Don't go wide too fast** — deep recon on 2–3 hosts beats shallow scans of 200.
- **JS recon and archive recon** regularly reveal hidden functionality that surface-level testing misses.
- **Keep consistent output naming** (`*_dirs.json`, `*_params.txt`, `*_urls.txt`) to reuse artifacts.
- **Timebox your recon** — set a limit, then switch to exploitation.

## References

- `methodology/web-recon-methodology.md`
- `methodology/javascript-endpoint-discovery.md`
- `tools/recon-tools.md`
- `techniques/` (all technique docs)
