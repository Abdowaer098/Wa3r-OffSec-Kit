# UmassCyberSec CTF: Bricktator Reactor Override - Full Write-up

## Challenge Overview
**Name:** Bricktator
**Description:** The Bricktator must be stopped! The mission all falls to you now. Infiltrate the Nuclear Control Center (NCC) and trigger a reactor override.
**Goal:** We need to bypass the NCC's authentication system and successfully approve a reactor override sequence which requires 5 high-clearance "YANKEE_WHITE" approvals.

---

## 1. Initial Reconnaissance
Upon analyzing the provided source code, we notice the application is a Spring Boot web app mapping several interesting endpoints. The primary objective revolves around `/command/override`, which initiates a reactor shutdown.

However, to authorize the shutdown, the `OverrideService` requires 5 approvals from sessions that possess the `YANKEE_WHITE` role. We are provided with the `bricktator` credentials (`bricktator:goldeagle`), which gives us the first `YANKEE_WHITE` session, leaving us needing 4 more.

Reviewing the Spring security configurations and controllers, we find that Spring Actuator is heavily exposed:
*   `/actuator/sessions`: Exposes active sessions for specific users.
*   `/actuator/accesslog`: Exposes a log of recent command access attempts.

## 2. Reconstructing Shamir's Secret Sharing
Looking at `SessionSeeder.java`, we discover how sessions are generated. The application seeds exactly 5000 valid sessions upon startup. The session IDs are mathematically generated using **Shamir's Secret Sharing**:
```java
String sessionId = "%05d-%08x".formatted(share.x(), share.y());
```
The polynomial evaluation is done over the prime `2147483647`. The secret polynomial has a `threshold = 3` (meaning `k=3`), so we need exactly 3 shares to reconstruct the entire polynomial and forge valid session IDs.

We can collect exactly three known shares by querying the `/actuator/sessions` endpoint for three pre-defined users:
1.  `John_Doe` (Index/x = 1)
2.  `Jane_Doe` (Index/x = 5)
3.  `bricktator` (Index/x = 5001)

By decoding the hex values of their session IDs, we get the `(x, y)` coordinate pairs. Using Lagrange interpolation over the finite field $\mathbb{Z}_{2147483647}$, we can write a Python script to reconstruct the polynomial and generate the mathematical ID for **all 5000 sessions**.

## 3. The Side-Channel Vulnerability
Although we can now generate all 5000 valid session IDs, we don't know which of these sessions have the `YANKEE_WHITE` role. `SessionSeeder` randomly picks 7 of the 5000 sessions to be `YANKEE_WHITE`, and testing them requires interacting with the `/command` endpoint.

Reviewing `CommandWorkFilter.java`, we find a critical side-channel vulnerability:
```java
if (request.getRequestURI().startsWith("/command")) {
    String rawId = resolveSessionId(request);
    if (rawId != null) {
        var session = sessionRepo.findById(rawId);
        if (session != null && "YANKEE_WHITE".equals(session.getAttribute("role"))) {
            accessLog.record(bcrypt.encode(rawId + PEPPER)); 
        }
    }
}
```
When a valid `YANKEE_WHITE` session attempts to access the `/command` endpoint, the application hashes the session ID using `BCryptPasswordEncoder` with a strength of 13.

**The Flaw:**
BCrypt with strength 13 is computationally expensive. It takes approximately **0.8 to 1.2 seconds** to compute synchronously on the request thread. If the session is *not* `YANKEE_WHITE`, the filter skips the BCrypt hashing entirely, resulting in a response time of **< 0.5 seconds**.

We successfully identified a **timing attack**. By iterating over our theoretically generated session IDs and measuring the response time of `GET /command`, we can accurately pinpoint the exact session IDs representing the remaining `YANKEE_WHITE` users without ever seeing their attributes directly.

*(Note: Initially, we attempted to use the `/actuator/accesslog` length as an oracle since it strictly records YANKEE_WHITE hits. However, an automated background bot organically generating traffic on the CTF instance disrupted our measurements and caused catastrophic false-positives. The timing attack completely bypassed the bot's background noise).*

## 4. The Final Exploit
With the puzzle pieces assembled, the final exploit `solve_bricktator_timing.py` works as follows:

1.  **Login:** Authenticate as `bricktator` and fetch the baseline `YANKEE_WHITE` cookies.
2.  **Fetch Shares:** Hit `/actuator/sessions` to grab the current state of shares for `John_Doe`, `Jane_Doe`, and `bricktator`.
3.  **Polynomial Generation:** Compute the Shamir Secret Sharing polynomial dynamically.
4.  **Timing Attack:** Use a multithreaded fast-scanner to hit `/command` with generated session IDs. We time the requests and filter those responding slower than `0.8` seconds.
5.  **Override Sequence:** Once 4 valid `YANKEE_WHITE` sessions are isolated, initiate the override via `POST /command/override` using the `bricktator` session.
6.  **Flag Capture:** Loop over the 4 found session IDs and submit them to `POST /override/{token}`. Upon the 5th collective approval, the server authorizes the sequence and dumps the flag in the HTML template.

### The Flag
```text
UMASS{stUx_n3T_a1nt_g0T_n0th1nG_0N_th15}
```
