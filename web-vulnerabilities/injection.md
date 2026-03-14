# Injection Vulnerabilities

## Summary
Injection vulnerabilities occur when untrusted data is sent to an interpreter as part of a command or query. The attacker's data tricks the interpreter into executing unintended commands or accessing data without authorization.

## Types

| Type | Interpreter | Impact |
|------|------------|--------|
| **SQL Injection** | Database (MySQL, PostgreSQL, MSSQL, etc.) | Data theft, auth bypass, RCE |
| **Command Injection** | Operating system shell | Full server compromise |
| **SSTI** | Template engine (Jinja2, Twig, Freemarker) | RCE via template expressions |
| **XXE** | XML parser | File read, SSRF, DoS |
| **LDAP Injection** | LDAP directory | Auth bypass, data disclosure |
| **NoSQL Injection** | MongoDB, CouchDB, etc. | Auth bypass, data theft |

## Key Resources in This Repository

### Techniques
- [SQL Injection](../techniques/sql-injection.md) — UNION, blind, time-based, OOB, WAF bypasses
- [Command Injection](../techniques/command-injection.md) — Operators, reverse shells, filter bypasses
- [Server-Side Template Injection](../techniques/server-side-template-injection.md) — Detection, engine fingerprinting, RCE
- [XXE Injection](../techniques/xxe-injection.md) — File read, SSRF, blind/OOB extraction
- [JSON Deserialization RCE](../techniques/json-deserialization-rce.md) — Java, Python, Node, PHP, .NET

### Case Studies
- [SSTI to Database Access (CTF)](../case-studies/case-study-ssti-rce-ctf.md)
- [XXE File Read (CTF)](../case-studies/case-study-xxe-ctf.md)

### Tools
- [Wordlists and Payload Lists](../tools/wordlists-and-payload-lists.md) — SSTI detection strings, XXE payloads

## Quick Reference

**Detection**: Inject special characters (`' " ; | {{ < $`) and look for errors, behavior changes, or reflected output.

**Impact**: Injection severity depends on the interpreter — SQLi can range from data disclosure to RCE. Command injection is almost always critical.
