# XSS (Cross-Site Scripting)

## Summary
XSS allows attackers to inject and execute client-side scripts in web pages viewed by other users. It is the most commonly reported web vulnerability class and can lead to session hijacking, credential theft, and malware distribution.

## Types

| Type | Description | Persistence |
|------|-------------|-------------|
| **Reflected** | Payload is in the URL/request and reflected in the response | None — requires victim to click a link |
| **Stored** | Payload is saved on the server and shown to other users | Persistent — affects all viewers |
| **DOM-based** | Payload is processed by client-side JavaScript, never touches the server | Depends on the JS sink |

## Key Resources in This Repository

### Techniques
- [XSS Techniques and Payloads](../techniques/xss-techniques.md) — Comprehensive guide covering contexts, bypass techniques, and categorized payloads.

### Scenarios
- [Reflected XSS Testing Workflow](../scenarios/reflected-xss-testing-workflow.md) — Step-by-step methodology from injection point discovery to impact escalation.

### Related Techniques (XSS chains)
- [CORS Misconfigurations](../techniques/cors-misconfigurations.md) — XSS + CORS = credential theft across origins.
- [Web Cache Poisoning](../techniques/web-cache-poisoning.md) — Self-XSS → cached stored XSS at CDN scale.
- [Client-Side Prototype Pollution](../techniques/prototype-pollution-client-side.md) — Prototype pollution → DOM XSS via gadgets.

### Tools
- [Wordlists and Payload Lists](../tools/wordlists-and-payload-lists.md) — XSS payloads organized by context and bypass type.

## Quick Reference

**Detection**: Inject a unique string, find where it's reflected, identify the HTML context.

**Impact escalation**: Don't just show `alert(1)` — demonstrate cookie theft, session hijacking, or phishing.
