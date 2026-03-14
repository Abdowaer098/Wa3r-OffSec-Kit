# JWT Attacks and Misconfigurations

## Overview
JSON Web Tokens (JWTs) are commonly used for stateless authentication and authorization. A JWT is a signed blob of JSON split into three base64url-encoded parts: header, payload, and signature. When implemented incorrectly, JWT validation logic can be abused to forge tokens, bypass authentication, or escalate privileges.

## When This Happens

You should look for JWT issues when:

- The application uses tokens starting with `eyJ` (typical JWT base64url prefix).
- Authentication is stateless (no server-side sessions; token is the source of truth).
- The server accepts JWTs from headers like `Authorization: Bearer ...` or cookies/local storage.
- Token headers and payloads are predictable or easily decodable.

Common risky patterns include:

- Disabled or weak signature verification (e.g., `alg: none`).
- Algorithm confusion between asymmetric (RS256) and symmetric (HS256) signing.
- Trusting embedded keys (JWK/JWKS) from untrusted sources.
- Failing to validate claims like `exp`, `aud`, `iss`.

## Recon / Detection

### 1. Identify and decode JWTs

Steps:

- Capture tokens from requests (headers, cookies, request bodies).
- Decode header and payload (without verifying the signature) using tools or online decoders.

Look at:

- `alg` in header (e.g., `HS256`, `RS256`, `none`).
- Claims in payload: `sub`, `role`, `admin`, `exp`, `aud`, etc.

### 2. Check for `alg` weaknesses

Key checks:

- Is `alg` set to `none` or can you change it to `none` and still be accepted?
- Does the server accept tokens if you modify `alg` from `RS256` to `HS256`?

If the server:

- Does not enforce an allow-list of algorithms.
- Uses the wrong key type for the algorithm.

…you may be able to sign your own tokens.

### 3. Look for embedded JWK/JWKS tricks

From your notes on the “Embedded JWK” attack:

- Some libraries will accept a JWK (JSON Web Key) inside the token or from external sources.
- If the server trusts keys that the attacker controls (e.g., from the header or a tampered JWKS), you can:
  - Provide your own public key.
  - Sign a token with the corresponding private key.
  - Have the server accept your forged token as valid.

Always inspect:

- `jwk`, `jku`, or `kid` fields in JWT headers.
- How the server fetches and validates keys.

## Exploitation Steps

### Attack 1: `alg: none` misuse

1. Decode the JWT header:
   ```json
   {
     "alg": "HS256",
     "typ": "JWT"
   }
   ```

2. Modify the header to:
   ```json
   {
     "alg": "none",
     "typ": "JWT"
   }
   ```

3. Remove the signature part (leave the trailing dot).
4. Rebuild the token and send it.

If the server accepts it, you can:
- Edit the payload (e.g., set `"role": "admin"`) and bypass authentication/authorization completely.

### Attack 2: Algorithm confusion (RS256 → HS256)

From your notes:

- The vulnerability appears when:
  - The server is configured for **RS256** (asymmetric).
  - Validation code allows switching to **HS256** (symmetric) and incorrectly uses the **public key as the HMAC secret**.

Steps:

1. Obtain the server’s public key (from:
   - A JWKS endpoint.
   - A PEM file embedded in the app.
   - Two JWTs and their structure, as you noted).

2. Change the header from:
   ```json
   { "alg": "RS256", "typ": "JWT" }
   ```
   to:
   ```json
   { "alg": "HS256", "typ": "JWT" }
   ```

3. Use the **public key** as the HMAC secret to sign a new token with your chosen payload (for example, admin role).

4. Send the forged token; if the server accepts it, you now control any claims (e.g., user id, role).

Impact:

- Full privilege escalation.
- Impersonation of any user.
- Bypass of authentication and access control.

### Attack 3: Embedded JWK / JKU abuse

If the header or server config allows it, you may:

1. Insert your own key into the header:
   ```json
   {
     "alg": "RS256",
     "typ": "JWT",
     "jwk": { ...your public key here... }
   }
   ```

2. Sign the token with the corresponding private key.
3. Rely on a misconfigured validator that trusts the embedded key rather than a fixed key set.

Result:

- You sign tokens as if you were the legitimate issuer.

## Example Payloads

- Modified header for algorithm confusion:
  ```json
  {
    "alg": "HS256",
    "typ": "JWT"
  }
  ```

- Example admin payload:
  ```json
  {
    "sub": "1234567890",
    "name": "attacker",
    "role": "admin",
    "admin": true
  }
  ```

Combine these with:

- Correct signature using:
  - The server’s public key as HMAC secret (for confusion).
  - Your own private key (for embedded JWK attacks).
- Or no signature at all (for `alg: none` misconfigurations).

## Tools

- **Burp Suite** (and JWT editor extensions) to:
  - Decode, edit, and re-sign tokens.
  - Automate testing of different `alg` and header combinations.
- **jwt.io** or CLI tools for manual experimentation and decoding.
- **Custom scripts** to:
  - Brute force or test different keys where appropriate.
  - Automate header/payload/signature recombination.

## Real-World Notes

- Your notes emphasize a critical point:
  - JWT is not “more secure than cookies” by default.
  - Security depends entirely on **correct algorithm handling and key management**.
- Any bug that lets an attacker control keys or algorithms effectively turns:
  - A **public key into a secret**.
  - Or **removes the need for a secret** altogether.
- Good defenses always:
  - Hard-code the expected `alg`.
  - Use a trusted key store (no attacker-controlled JWK/JWKS).
  - Validate all critical claims (`exp`, `iss`, `aud`, etc.).

## References

- `methodology/web-recon-methodology.md`
- `tools/recon-tools.md`

