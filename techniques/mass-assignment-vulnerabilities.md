# Mass Assignment Vulnerabilities

## Overview
Mass assignment (also called overposting) happens when a backend automatically binds all user-supplied fields into server-side objects without filtering. This allows an attacker to set internal properties that should never be controlled from the client, such as prices, discount percentages, roles, or flags like `is_admin`.

## When This Happens

You are likely facing mass assignment when:

- The application uses frameworks that support automatic model binding (e.g., Rails, Laravel, Django REST, many ORMs).
- API requests accept rich JSON bodies that mirror database models.
- You see internal-looking fields in client-side JSON (for example: `item_price`, `percentage`, `role`, `is_admin`).
- Changes to those fields directly affect business logic without additional server-side validation.

Typical vulnerable pattern (pseudocode):

```python
# Vulnerable style
order = Order(request.json)       # or update(order, request.body)
db.save(order)
```

Instead of explicitly allow-listing safe fields.

## Recon / Detection

### 1. Identify JSON-heavy or object-like endpoints

Look for:

- `/api/order`, `/api/user`, `/api/profile`, `/checkout`, `/cart`, `/subscription`.
- HTTP methods: `POST`, `PUT`, `PATCH` with JSON bodies.

Use:

- Burp history and `Replacer`/`Logger++`.
- Your JS recon (`javascript-endpoint-discovery`) and WaerJScan output to find candidate endpoints and parameters.

### 2. Compare client-visible fields vs. server behavior

Ask:

- Which fields should be **user-controlled** (e.g., `quantity`, `product_id`)?
- Which fields are **server-controlled** (e.g., `item_price`, `discount`, `is_admin`, `status`, `verified`)?

Red flags:

- The client sends full objects including sensitive properties.
- Modifying a “server-controlled” field in the request changes the outcome.

### 3. Bruteforce hidden fields

If the JSON only shows some fields, try:

- Adding common internal fields:
  - `role`, `roles`, `is_admin`, `is_staff`, `is_superuser`
  - `price`, `item_price`, `discount`, `percentage`, `total`
  - `status`, `state`, `approved`, `verified`
- Copying fields you see in responses or JS objects into your request body.

Monitor:

- Response body changes.
- Behavior changes in the app (e.g., price, access, state changes).

## Exploitation Steps

### Example scenario: price and discount manipulation

Observed request:

```json
{
  "chosen_products": [
    {
      "product_id": "1",
      "quantity": 1,
      "item_price": 0
    }
  ],
  "chosen_discount": {
    "percentage": 100
  }
}
```

Problem:
- Client is allowed to control `item_price` and `percentage`.
- Backend likely binds the whole JSON into an `Order` or `Cart` object directly.

Attack flow:

1. Intercept the request in Burp.
2. Set `item_price` to `0` or manipulate `percentage` to `100`.
3. Forward the request and confirm:
   - Total is reduced or becomes zero.
   - Order is accepted and processed.

Impact:
- Free or heavily discounted products.
- Potential abuse of “internal” discounts or staff-only pricing.

### Example scenario: privilege escalation

If you find or guess a field like:

```json
{
  "email": "user@target.com",
  "is_admin": true
}
```

or:

```json
{
  "role": "admin"
}
```

And the backend mass-assigns it into the user object, you may:

- Upgrade your role to administrator.
- Access admin-only panels and APIs.

## Example Payloads

- Price/discount:
  ```json
  {
    "product_id": "1",
    "quantity": 1,
    "item_price": 0,
    "discount": 100
  }
  ```

- Role/privilege:
  ```json
  {
    "email": "attacker@target.com",
    "role": "admin",
    "is_admin": true
  }
  ```

- Status/approval:
  ```json
  {
    "status": "approved",
    "verified": true
  }
  ```

Combine these ideas with parameter names discovered via:
- `arjun`
- WaerJScan (`parameters.txt`)
- Your frontend/JS recon.

## Tools

- **Burp Suite**: Interception, repeater, and param bruteforcing.
- **Arjun**: Discover hidden parameters that may map to internal fields.
- **WaerJScan / JS recon**: Extract field names and structures from JS and responses.
- **Custom wordlists**: Use your generated wordlists for typical sensitive property names.

## Real-World Notes

- Mass assignment is not only a technical issue; it is a **business logic vulnerability** because it breaks rules around who controls which fields.
- Good writeups clearly separate:
  - **User-controlled fields** (legitimate).
  - **Server-controlled fields** (should never be writable by clients).
  - The exact impact (free items, role escalation, bypassing manual approvals).
- Many frameworks provide explicit protection (e.g., allow-lists/whitelists), but teams forget to configure them.

## References

- `methodology/web-recon-methodology.md`
- `tools/recon-tools.md`
