# File Upload and Deserialization

## Summary
File upload and deserialization vulnerabilities allow attackers to execute arbitrary code on the server by uploading malicious files or exploiting unsafe object deserialization. Both can lead directly to Remote Code Execution.

## Types

| Type | Attack Vector | Impact |
|------|--------------|--------|
| **Webshell upload** | Upload PHP/JSP/ASP file, access via URL | OS command execution |
| **Extension bypass** | Use alternative extensions (.php5, .phtml) | Webshell execution |
| **Content-Type bypass** | Spoof MIME type headers | Bypass client-side validation |
| **Magic byte bypass** | Prepend valid image headers to shell | Bypass server-side validation |
| **SVG upload** | Embed XSS or XXE in SVG XML | Stored XSS, file read |
| **Deserialization** | Send malicious serialized objects | RCE via gadget chains |

## Key Resources in This Repository

### Techniques
- [File Upload Vulnerabilities](../techniques/file-upload-vulnerabilities.md) — Extension/content-type/magic byte bypasses, webshells, SVG attacks, race condition uploads
- [JSON Deserialization RCE](../techniques/json-deserialization-rce.md) — Java, Python, Node, PHP, .NET unsafe deserialization
- [XXE Injection](../techniques/xxe-injection.md) — SVG-based XXE via file upload

### Tools
- [Wordlists and Payload Lists](../tools/wordlists-and-payload-lists.md)

## Quick Reference

**Detection**: Upload test files with various extensions. Check if uploaded files are accessible and executable. Test with polyglot files (valid image + embedded code).

**Impact**: Successful file upload → webshell = full server compromise. Deserialization RCE = critical.
