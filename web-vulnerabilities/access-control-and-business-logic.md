# Access Control and Business Logic

## Summary
Access control vulnerabilities occur when the application fails to enforce authorization — allowing users to access resources, functions, or data they shouldn't. Business logic flaws exploit the intended workflow of the application by abusing assumptions in the design.

## Types

| Type | Description | Example |
|------|-------------|---------|
| **IDOR** | Direct object reference without authorization check | `/api/users/123` → change to `/api/users/124` |
| **Horizontal priv esc** | Same role, different user's data | User A reads User B's profile |
| **Vertical priv esc** | Low role accessing high role functions | User → Admin functionality |
| **Mass Assignment** | Setting unauthorized fields via API | Adding `role=admin` to a request |
| **Logic Flaws** | Abusing application workflow assumptions | Skipping payment step in checkout |
| **Race Conditions** | Exploiting timing between check and action | Applying a coupon 50 times simultaneously |

## Key Resources in This Repository

### Techniques
- [IDOR](../techniques/idor-insecure-direct-object-reference.md) — Horizontal/vertical escalation, UUID techniques, testing checklist
- [Mass Assignment](../techniques/mass-assignment-vulnerabilities.md) — Field injection for privilege escalation
- [Race Conditions](../techniques/race-conditions.md) — Timing attacks on financial operations and state changes
- [Open Redirect](../techniques/open-redirect.md) — URL redirect abuse for phishing and token theft

### Case Studies
- [Client-Side Checkout Password Leak](../case-studies/case-study-checkout-password-leak.md) — Hardcoded password in JavaScript
- [WordPress API Enumeration](../case-studies/case-study-recon-wordpress-api-enumeration.md) — Unauthorized user data access

### Scenarios
- [Logic Flaw: Unauthorized Checkout](../scenarios/logic-flaw-unauthorized-checkout-workflow.md) — Client-side authorization bypass

### Methodology
- [Exploitation Methodology](../methodology/exploitation-methodology.md) — Systematic exploitation approach
- [Privilege Escalation Checklist](../methodology/privilege-escalation-checklist.md) — Web, Linux, Windows, AWS privesc

## Quick Reference

**Detection**: Always test with two accounts. Change IDs, add hidden parameters, skip workflow steps, send parallel requests.

**Impact**: IDOR is the #1 most common bug bounty finding. Write IDORs (PUT/DELETE) are higher severity than read IDORs.
