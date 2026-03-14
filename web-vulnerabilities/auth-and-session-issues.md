# Authentication and Session Issues

## Summary
Authentication and session management vulnerabilities allow attackers to bypass login mechanisms, hijack user sessions, escalate privileges, or take over accounts. This category covers password reset abuse, JWT attacks, session handling flaws, and credential-related logic bugs.

## Key Resources in This Repository

### Techniques
- [Password Reset Abuse](../techniques/password-reset-abuse.md) — Comprehensive testing methodology for password reset flows: token leakage, host header poisoning, email injection, brute force, IDOR.
- [JWT Attacks and Misconfigurations](../techniques/jwt-attacks-and-misconfigurations.md) — Algorithm confusion, embedded JWK attacks, RS256→HS256 downgrades, key injection.
- [Mass Assignment](../techniques/mass-assignment-vulnerabilities.md) — Privilege escalation via role/admin field injection.

### Case Studies
- [Client-Side Checkout Authentication Bypass](../case-studies/case-study-checkout-password-leak.md) — Hardcoded passwords in JavaScript, client-side auth gate bypass.
- [WordPress API Enumeration](../case-studies/case-study-recon-wordpress-api-enumeration.md) — User enumeration and API surface mapping revealing auth weaknesses.
- [SSTI to Database Access](../case-studies/case-study-ssti-rce-ctf.md) — Template injection leading to admin credential extraction.

### Scenarios
- [Logic Flaw: Unauthorized Checkout](../scenarios/logic-flaw-unauthorized-checkout-workflow.md) — Generic workflow for exploiting client-side authentication gates.

## Common Vulnerability Patterns

| Pattern | Test for | Impact |
|---------|----------|--------|
| Weak password reset tokens | UUID v1 prediction, short OTPs | Account takeover |
| Missing session invalidation | Sessions persist after password change | Persistent access |
| Client-side auth gates | JS-only validation, hardcoded credentials | Authorization bypass |
| JWT algorithm confusion | `alg: none`, RS256→HS256 | Authentication bypass |
| Role manipulation | Mass assignment of `is_admin`, `role` fields | Privilege escalation |
| IDOR on user endpoints | Sequential IDs, predictable tokens | Data access |

## Quick Reference

**Always test**: Token expiration, session behavior after password change, host header influence on reset links, JWT algorithm flexibility, hidden role/admin parameters.
