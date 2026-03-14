# Case Study: Client-Side Authentication Bypass on Checkout

## Overview
This case study documents the discovery of a client-side authentication bypass on an e-commerce checkout page. The application required a "data entry password" to authorize purchases, but the authentication logic was implemented entirely in JavaScript — with passwords hardcoded in the page source code.

## Target Context (anonymized)

- **Platform**: WordPress + WooCommerce e-commerce site.
- **Feature**: Investment packages / course purchase flow.
- **Access**: No account required to add items to cart and reach checkout.

## Vulnerability

**Type**: Client-Side Trust / Hardcoded Credentials in JavaScript

The checkout page contained a password field labeled as "data entry authorization." This gate was intended to restrict who could complete purchases. However:

1. The password validation was performed entirely in client-side JavaScript.
2. The actual passwords were **hardcoded in the page source**.
3. No server-side validation occurred — once the JS check passed, the order was submitted directly.

## Recon and Discovery

### 1. Reaching the checkout without an account
- Navigated to the investment packages page (no authentication required).
- Added a package to the cart.
- Proceeded to checkout — the cart functioned without any account.

### 2. Encountering the client-side gate
- The checkout form included a "data entry password" field.
- Upon entering an incorrect value, the form rejected the submission instantly — **no network request was made**.
- This immediate rejection (with no server round-trip) strongly indicated client-side validation.

### 3. Inspecting the source code
- Right-click was disabled on the page (a superficial protection).
- Used `Ctrl+U` to view the page source.
- Found the password validation logic in JavaScript, including all valid passwords hardcoded as string literals.

## Exploitation

### Attack flow

1. View page source (`Ctrl+U` to bypass right-click block).
2. Search for the password validation function.
3. Extract the hardcoded data-entry passwords.
4. Enter a valid password in the checkout form.
5. Complete the purchase as an unauthorized user.

### Alternatively (bypass without the password)
Since validation is client-side only:
- Delete the password check function via browser DevTools.
- Or modify the JavaScript to always return `true`.
- Or submit the form directly via `fetch()` / curl, skipping the JS entirely.

## Impact

| Impact area | Description |
|------------|-------------|
| **Unauthorized purchases** | Anyone can complete checkout without proper authorization |
| **Credential leak** | Data entry passwords exposed to all visitors in page source |
| **Business logic bypass** | The authorization gate is completely ineffective |
| **No account required** | The entire flow from browse → cart → checkout → purchase works unauthenticated |

## Lessons Learned

1. **Never trust client-side authentication** — any security check that runs in the browser can be bypassed by the user.
2. **Hardcoded credentials in JavaScript are visible to everyone** — even with obfuscation, the values can be extracted.
3. **Right-click disable is not a security control** — `Ctrl+U`, DevTools, `curl`, and browser extensions all bypass it trivially.
4. **Critical business logic must be enforced server-side** — the checkout authorization should have been validated by the backend before processing the order.
5. **"No network request" on validation failure** is a strong indicator that the check is client-side only — always investigate further.

## References

- `techniques/mass-assignment-vulnerabilities.md` (related business logic attack)
- `scenarios/logic-flaw-unauthorized-checkout-workflow.md` (generic workflow)
- `methodology/web-recon-methodology.md`
