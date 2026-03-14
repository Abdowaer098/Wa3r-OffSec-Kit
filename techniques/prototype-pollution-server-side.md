# Server-Side Parameter Pollution (SSPP)

## Overview
Server-Side Parameter Pollution (SSPP) occurs when user input is embedded into a server-side API request without proper encoding or validation. The attacker cannot see the internal request directly, but by injecting special characters (`&`, `#`, `../`, etc.), they can add, remove, or override parameters and path segments in the backend request — potentially leading to privilege escalation, data disclosure, or account takeover.

The key mental model:
```
User → Website → Internal API
                  ↑ you control parts of this
```

You are not attacking the endpoint in front of you — you are trying to understand and manipulate what the server does behind the scenes.

## When This Happens

You are likely facing SSPP when:

- User input is forwarded to an internal/backend API.
- You see unusual error messages when adding special characters (`&`, `#`, `../`) to parameters.
- The response changes based on characters that should not affect a simple value.
- The application uses a microservices architecture where the frontend proxies requests to backend services.

## Recon / Detection

### 1. Identify internal API forwarding
Look for signs that user input is passed to another service:
- Error messages referencing internal endpoints or parameter names.
- Response discrepancies when adding URL-encoded special characters.
- API responses that seem to come from a different system than the frontend.

### 2. Test for parameter injection
Try appending encoded special characters to normal input values:

| Character | Encoded | Purpose |
|-----------|---------|---------|
| `&` | `%26` | Add a new parameter |
| `#` | `%23` | Truncate the rest of the query |
| `=` | `%3d` | Change parameter values |
| `/` | `%2f` | Path traversal |
| `../` | `%2e%2e%2f` | Navigate up in REST paths |

### 3. Observe response changes
- Does adding `%26x=y` change the response? → Parameter injection confirmed.
- Does adding `%23` remove expected data? → You can truncate the internal query.
- Does `../` produce different content? → Path traversal in REST URLs.

## Exploitation Steps

### Type 1: Query String SSPP

**Internal request pattern:**
```
/api/users/search?name={user_input}&field=email
```

**Attack flow:**
1. Test normal input → understand baseline response.
2. Inject `%26x=y` → confirm parameter injection.
3. Inject `%23` → truncate parameters after your input (removes `&field=email`).
4. Discover internal parameter names (e.g., `field`) via brute force.
5. Override the `field` parameter: `input%26field=reset_token%23`
6. The internal request becomes: `/api/users/search?name=input&field=reset_token`
7. The server returns the user's password reset token.
8. Use the token to reset the admin's password → **account takeover**.

**Injection tools:**

| Symbol | Function |
|--------|----------|
| `%26` (`&`) | Add new parameter |
| `%23` (`#`) | Cut off remaining query |
| `=` | Set/change values |

### Type 2: REST Path SSPP

**Internal request pattern:**
```
/api/internal/v1/users/{username}/field/{field}
```

The `{username}` comes from user input and is placed directly in the URL path.

**Attack flow:**
1. Confirm input is in the URL path (not query string).
2. Inject `./` — if the response is the same, path processing is happening.
3. Inject `../` — attempt path traversal within the API.
4. Try to reach API documentation: `../../../openapi.json`
5. Discover hidden endpoints from the API spec.
6. Inject path segments: `username/../admin/field/reset_token`
7. Try different API versions: change `/v1/` to `/v2/`
8. Extract sensitive data (e.g., `passwordResetToken`).
9. Use the token → **admin account takeover**.

**Injection tools:**

| Technique | Purpose |
|-----------|---------|
| `#` or `?` | Break the path |
| `./` | Test same-level path |
| `../` | Path traversal |
| `openapi.json` | Discover API documentation |

## Comparison: Query SSPP vs REST SSPP

| | Query SSPP | REST SSPP |
|---|---|---|
| **Injection point** | Query parameters | URL path |
| **Key characters** | `&` `#` `=` | `../` `/` `?` |
| **Difficulty** | Medium | High |
| **Target** | Parameter values | Routes and endpoints |
| **Discovery** | Parameter brute force | Path traversal + API docs |

## Example Payloads

**Query string SSPP:**
```
# Add parameter and truncate
username=admin%26field=reset_token%23

# Override existing parameter
search=test%26role=admin%23
```

**REST path SSPP:**
```
# Path traversal to API docs
username=../../openapi.json

# Access different resource
username=admin/field/passwordResetToken

# API version switching
username=../../../v2/users/admin/field/token
```

## Quick Detection Signs

If you notice:
- ✅ User input goes to a backend request
- ✅ Unusual error messages with special characters
- ✅ Response changes with `&`, `#`, or `../`

→ High probability of SSPP.

## Tools

- **Burp Suite**: Intercept, modify, and replay requests with injected characters.
- **Param Miner**: Discover hidden internal parameters.
- **ffuf**: Brute force internal parameter names and path segments.
- **Custom scripts**: Automate path traversal enumeration.

## Impact

- **Account takeover** via leaked reset tokens
- **Data disclosure** of internal API responses
- **Access to internal APIs** not meant for external users
- **Privilege escalation** by overriding role/permission parameters

## Real-World Notes

- The root cause is always the same: the developer built the backend request using string concatenation (`url = "/api/users/" + username`) instead of proper encoding, validation, or parameterized queries.
- SSPP is underreported but common in microservice architectures where the frontend acts as a proxy.
- Always think: *"What is the server talking to behind the scenes?"* — once you understand the internal request shape, you can start controlling it.

## Mitigation

- Encode all user input before embedding it in internal requests.
- Use allowlists for permitted characters.
- Never build URLs via string concatenation.
- Separate internal API input from user-controlled input.

## References

- `techniques/mass-assignment-vulnerabilities.md`
- `methodology/web-recon-methodology.md`
- `tools/recon-tools.md`
