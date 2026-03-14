# Scenario: Logic Flaw — Unauthorized Checkout via Client-Side Auth Bypass

## Overview
A workflow for identifying and exploiting client-side authentication gates in e-commerce checkout flows. The pattern: a form field requires a password or code, but validation happens only in JavaScript — meaning it can be bypassed by inspecting the source code, modifying JS, or replaying requests directly.

## When This Happens
- An e-commerce site has a checkout, purchase, or approval flow with an extra authorization step.
- A password, PIN, or access code is required to complete a transaction.
- Form submission is blocked instantly (no network request) when the wrong code is entered — indicating client-side validation.

## Recon / Detection

### Step 1: Trigger the gate
- Add items to cart, proceed to checkout.
- Identify any extra form fields (access code, data entry password, PIN).

### Step 2: Check for client-side validation
- Enter an incorrect value and observe:
  - **Instant rejection with no network activity** → client-side.
  - **Server round-trip before rejection** → server-side (harder to bypass).
- Open DevTools Network tab and confirm no request is sent on failure.

### Step 3: Inspect the source code
- `Ctrl+U` (View Source) or `Ctrl+Shift+I` (DevTools) → search for the validation logic.
- Search for keywords: `password`, `pin`, `access`, `validate`, `check`, `verify`.
- Look for hardcoded values, comparison strings, or validation functions.

## Exploitation Steps

### Option A: Extract the password
1. Find the validation function in the page source.
2. Extract the hardcoded credentials.
3. Enter them in the form and proceed normally.

### Option B: Bypass the validation
1. Open DevTools Console.
2. Override the validation function:
   ```javascript
   // Find the function name from source inspection
   validatePassword = function() { return true; }
   ```
3. Submit the form.

### Option C: Direct request replay
1. Capture a valid checkout request (submit the form with DevTools Network tab open).
2. Replay it in Burp or curl, completely skipping the JS validation:
   ```bash
   curl -X POST https://target.com/checkout \
     -H "Content-Type: application/json" \
     -d '{"items": [...], "payment": {...}}'
   ```

### Additional checks
- Can checkout be completed **without any account**?
- Can the price or quantity be modified in the request body? (→ mass assignment)
- Does the server validate the total price, or trust the client?

## Example Payloads
```javascript
// Console override
document.querySelector('#passwordForm').onsubmit = function() { return true; }

// Or delete the validation element entirely
document.querySelector('#accessCodeField').remove()
```

## Tools
- **Browser DevTools**: Source inspection, console overrides, network monitoring.
- **Burp Suite**: Intercept and modify checkout requests.
- **curl**: Direct API calls bypassing the frontend entirely.

## Real-World Notes
- This is a **business logic vulnerability**, not a technical exploit — the impact is unauthorized transactions.
- Right-click disable, JS obfuscation, and minification are NOT security controls.
- Always check if sensitive authorization happens server-side by intercepting at the network level.

## References
- `case-studies/case-study-checkout-password-leak.md`
- `techniques/mass-assignment-vulnerabilities.md`
