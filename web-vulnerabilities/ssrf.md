# SSRF (Server-Side Request Forgery)

## Summary
SSRF occurs when an attacker can make the server initiate HTTP requests to arbitrary destinations — including internal services, cloud metadata endpoints, and the server itself. It can lead to internal network scanning, data exfiltration, and cloud account compromise.

## Key Resources in This Repository

### Scenarios
- [SSRF via XML-RPC Pingback Workflow](../scenarios/ssrf-via-xmlrpc-pingback-workflow.md) — Step-by-step workflow for testing WordPress XML-RPC SSRF.

### Case Studies
- [Case Study: SSRF via WordPress XML-RPC](../case-studies/case-study-ssrf-xmlrpc-wordpress.md) — Real-world discovery and exploitation of `pingback.ping` SSRF.

### Related Techniques
- [XXE Injection](../techniques/xxe-injection.md) — XXE can enable SSRF by making the XML parser fetch attacker-specified URLs.
- [Server-Side Parameter Pollution](../techniques/prototype-pollution-server-side.md) — SSPP can redirect internal API calls to unintended endpoints.

### Tools
- [Recon Tools](../tools/recon-tools.md) — Tools for discovering SSRF-vulnerable endpoints.

## Common SSRF Targets

| Target | URL | Purpose |
|--------|-----|---------|
| AWS metadata | `http://169.254.169.254/latest/meta-data/` | IAM credentials, instance info |
| GCP metadata | `http://metadata.google.internal/` | Service account tokens |
| Azure metadata | `http://169.254.169.254/metadata/instance` | Instance metadata |
| Internal services | `http://127.0.0.1:PORT` | Access local services |
| Internal network | `http://192.168.x.x` | Scan internal hosts |

## Quick Reference

**Detection**: Any feature where the server fetches a URL (webhooks, image import, PDF generation, URL preview, XML parsing).

**Impact**: Cloud metadata access = critical. Internal scanning = high. External callback only = medium.
