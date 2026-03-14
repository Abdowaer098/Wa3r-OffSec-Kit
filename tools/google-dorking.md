# Google Dorking for Bug Bounty

## Overview
Google Dorking (Google Hacking) uses advanced search operators to find security programs, exposed files, admin panels, and information disclosure across the web. This document provides a comprehensive dork collection organized by purpose.

## Finding Bug Bounty Programs

### Disclosure policy pages
```
inurl:/responsible disclosure
inurl:/responsible-disclosure/ reward
inurl:/responsible-disclosure/ bounty
inurl:/responsible-disclosure/ swag
inurl:responsible-disclosure-policy
inurl:vulnerability-disclosure-policy reward
inurl:reporting-security-issues
inurl:security-policy.txt ext:txt
```

### Security.txt files
```
inurl:security.txt
inurl:/.well-known/security ext:txt
inurl:/.well-known/security ext:txt intext:hackerone
inurl:/.well-known/security ext:txt -hackerone -bugcrowd -synack -openbugbounty
/trust/report-a-vulnerability
site:*/security.txt "bounty"
inurl:/security ext:txt "contact"
```

### Platform-specific
```
"powered by bugcrowd" -site:bugcrowd.com
"powered by hackerone" -site:hackerone.com
"powered by synack"
"submit vulnerability report"
"submit vulnerability report" | "powered by bugcrowd" | "powered by hackerone"
inurl:private bugbountyprogram
```

### General bounty discovery
```
inurl:/bug bounty
inurl:/security
inurl:security "reward"
responsible disclosure hall of fame
intext:responsible disclosure
intext:"we take security very seriously"
intext:Vulnerability Disclosure
"security vulnerability" "report"
"If you believe you've found a security vulnerability"
"If you find a security issue" "reward"
intext:bounty inurl:/security
intext:we offer a bounty
intext:responsible disclosure bounty
intext:"BugBounty" and intext:"BTC" and intext:"reward"
```

### By currency / region
```
inurl:"bug bounty" and intext:"$" and inurl:/security
inurl:"bug bounty" and intext:"€" and inurl:/security
inurl:"bug bounty" and intext:"INR" and inurl:/security
inurl:bug bounty intext:"rupees"
inurl:bug bounty intext:"₹"
buy bitcoins "bug bounty"
```

### By country TLD
```
responsible disclosure r=h:com
responsible disclosure r=h:nl
responsible disclosure r=h:uk
responsible disclosure r=h:eu
site:*.gov.* "responsible disclosure"
site:*.edu intext:security report vulnerability
site:*.nl responsible disclosure
site:*.de inurl:bug inurl:bounty
site:*.uk intext:security report reward
site:*.cn intext:security report reward
site:*.br responsible disclosure
site:*.at responsible disclosure
site:*.be responsible disclosure
site:*.au responsible disclosure
```

### Hall of fame / reward types
```
responsible disclosure $50
responsible disclosure white hat
white hat program
responsible disclosure swag r=h:com
responsible disclosure bounty r=h:nl
responsible disclosure reward r=h:uk
"responsible disclosure" intext:"you may be eligible for monetary compensation"
```

## Target Reconnaissance Dorks

### Admin panels and sensitive files
```
site:target.com inurl:admin
site:target.com inurl:login
site:target.com inurl:dashboard
site:target.com filetype:env
site:target.com filetype:config
site:target.com filetype:log
site:target.com filetype:sql
site:target.com filetype:bak
site:target.com filetype:old
```

### Sensitive document types
```
site:target.com (ext:doc OR ext:docx OR ext:pdf OR ext:xls OR ext:xlsx OR ext:csv)
site:target.com (ext:bak OR ext:old OR ext:backup OR ext:zip OR ext:rar OR ext:7z OR ext:tar OR ext:gz)
site:target.com (ext:txt OR ext:log OR ext:cfg OR ext:conf)
```

### Error pages and debug info
```
site:target.com intext:"sql syntax"
site:target.com intext:"stack trace"
site:target.com intext:"php error"
site:target.com intitle:"index of /"
```

### Security infrastructure
```
site:help.target.com inurl:bounty
site:support.target.com intext:security report reward
site:security.target.com inurl:bounty
site:*.*.target.com inurl:bug inurl:bounty
```

### Bug bounty scope expansion
```
site:*.target.com
site:*.*.target.com
inurl:target.com -www
```

## WordPress-Specific Dorks

### User enumeration
```
site:target.com inurl:/wp-json/wp/v2/users
site:target.com /author-sitemap.xml
```

### Sensitive files
```
site:target.com /wp-content/debug.log
site:target.com /wp-config.php
site:target.com filetype:sql "wp_users"
```

### Login and registration
```
site:target.com /wp-login.php?action=register
site:target.com /wp-admin/login.php
```

## Usage Tips

1. **Use these in Google search** — paste directly into the search bar.
2. **Combine operators** — chain `site:`, `inurl:`, `intext:`, `filetype:` for precision.
3. **Rate limiting** — Google will CAPTCHA you if you dork too fast. Use a VPN and space out searches.
4. **Automation** — Tools like `dorkbot`, `pagodo`, or custom scripts can automate dork searches.
5. **Save results** — Export findings to a structured file (`dork_results.csv`) for tracking.

## References

- `methodology/bug-bounty-playbook.md`
- `methodology/web-recon-methodology.md`
- `tools/recon-tools.md`
