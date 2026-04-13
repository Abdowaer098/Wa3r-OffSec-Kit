# UmassCyberSec CTF: Bricktator Reactor Override v2 - Full Write-up

## Challenge Overview
**Name:** Bricktator v2
**Goal:** Bypass the Nuclear Control Center's (NCC) hardened authentication system and trigger a reactor override. This requires 5 "YANKEE_WHITE" approvals.
**Initial Access:** Credentials `bricktator:goldeagle`.

---

## 1. Hardening in v2
In this version, the infrastructure was significantly hardened:
*   **Actuator Lockdown:** The `/actuator/**` endpoints (sessions, health, etc.) are now protected by Spring Security. Access requires at least the `Q_CLEARANCE` role.
*   **Session Fixation Protection:** While `sessionFixation().none()` was used in `SessionConfig`, the `SessionSuccessHandler` forces the session ID into a specific format `%05d-%08x` to match the Shamir's Secret Sharing scheme.
*   **Environmental Noise:** Increased concurrent traffic and background bots made simple timing/Oracle attacks more difficult due to network jitter.

---

## 2. Technical Vulnerabilities

### A. Shamir's Secret Sharing Reconstruction
The application uses **Shamir’s Secret Sharing** to generate session IDs. These IDs are coordinates $(x, y)$ evaluated on a private polynomial over the prime $P = 2147483647$.
The polynomial has a threshold $k=3$, meaning we need 3 points to reconstruct it entirely.

The `SessionSeeder` assigns specific indices:
- `John_Doe`: $x = 1$
- `Jane_Doe`: $x = 5$
- `bricktator`: $x = 5001$

By logging in as `bricktator`, we gain the `YANKEE_WHITE` role which allows us to query `/actuator/sessions` for other users. We retrieved the points for `John_Doe` and `Jane_Doe`, and used our own session ID as the third point.

Using **Lagrange Interpolation** in the finite field $\mathbb{Z}_{P}$, we can determine the polynomial:
$$L(x) = \sum_{j=0}^{k-1} y_j \prod_{m=0, m \neq j}^{k-1} \frac{x - x_m}{x_j - x_m} \pmod P$$

### B. BCrypt Side-Channel (Timing Attack)
`CommandWorkFilter.java` contains a critical timing vulnerability:
```java
if (session != null && "YANKEE_WHITE".equals(session.getAttribute("role"))) {
    accessLog.record(bcrypt.encode(rawId + PEPPER)); 
}
```
If a session ID belongs to a `YANKEE_WHITE` user, the server performs a `bcrypt.encode()` with a strength of 13. This takes **~1.0 seconds**. If the user is NOT `YANKEE_WHITE`, the hash is skipped, resulting in a **~0.3 second** response time.

---

## 3. Exploit Execution

### Step 1: Authentication & Data Gathering
We authenticate as `bricktator` and use the session to query `/actuator/sessions`.
*   **Point 1:** `(1, 1502978480)`
*   **Point 2:** `(5, 744410102)`
*   **Point 3:** `(5001, 1679865517)`

### Step 2: Session Forgery
We iterate through all possible indices $x \in [1, 5000]$ and calculate the corresponding $y$ value using our reconstructed polynomial. We then format these into Base64-encoded `SESSION` cookies.

### Step 3: Timing Attack with Verification
To handle network noise, we implemented a **Verification Loop**:
1. Scan sessions in parallel (20 threads).
2. If a session takes $> 0.8s$, flag it.
3. Re-test flagged sessions 3 times. If the *minimum* time is still $> 0.8s$, confirm it as `YANKEE_WHITE`.

### Step 4: Override Sequence
Once 4 new `YANKEE_WHITE` sessions are identified:
1. `POST /command/override` as `bricktator` to get an **Override Token**.
2. Loop through the 4 found sessions and `POST /override/{token}` using their respective cookies.
3. On the 5th approval (the last one), the server returns the flag.

---

## 4. The Flag
```text
UMASS{stUx_n3T_a1nt_g0T_n0th1nG_0N_th15_v2!!!randomNoiseAndStuff}
```

---

## 5. Artifacts
The full automated solve script is available locally at `solve_v3.py`.
