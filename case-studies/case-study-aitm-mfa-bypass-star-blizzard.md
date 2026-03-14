# Case Study: MFA Bypass via Adversary-in-the-Middle (Star Blizzard APT Emulation)

## Overview
This case study documents the emulation of **Star Blizzard** (formerly SEABORGIUM/Callisto Group), a Russian-linked cyber espionage group targeting academia, governments, and NGOs across the UK, USA, and NATO-affiliated entities. The simulation demonstrates their Adversary-in-the-Middle (AiTM) technique to **bypass Multi-Factor Authentication (MFA)** on Microsoft Azure/Office 365 by intercepting authenticated session cookies.

The key insight: **MFA protects the authentication process, but not the session token that results from it.** If you can steal the session cookie, MFA becomes irrelevant.

## MITRE ATT&CK Mapping

| Tactic | Technique |
|--------|-----------|
| **Initial Access (TA0001)** | T1566.002 — Phishing: Spearphishing Link |
| **Credential Access (TA0006)** | T1557 — Adversary-in-the-Middle |
| **Defense Evasion (TA0005)** | T1550.004 — Use Alternate Authentication Material: Web Session Cookie |
| **Collection (TA0009)** | T1530 — Data from Cloud Storage Object |

## How AiTM Works

```
┌──────────┐        ┌───────────────────┐        ┌──────────────────┐
│  Victim  │───────▶│  Attacker Proxy   │───────▶│  Real Azure IdP  │
│ (Browser)│◀───────│  (Evilginx/Modl)  │◀───────│  (login.micro... │
└──────────┘        └───────────────────┘        └──────────────────┘
                           │
                     Intercepts:
                     ✓ Username
                     ✓ Password
                     ✓ MFA token
                     ✓ Session cookies
```

1. Attacker sets up a **transparent reverse proxy** (e.g., Evilginx2) that mirrors the real Azure login page.
2. Victim receives a phishing email with a link to the attacker's proxy (disguised with a convincing domain).
3. Victim enters credentials on the proxy → proxy forwards them to real Azure.
4. Azure prompts for MFA → victim completes MFA → proxy intercepts the response.
5. Azure issues session cookies (`ESTSAUTH`, `ESTSAUTHPERSISTENT`) → proxy captures them.
6. Attacker uses the stolen cookies to access Azure Portal **as the victim**, completely bypassing MFA.

## Attack Flow

### Phase 1: Infrastructure Setup

**Evilginx2 Configuration:**
The attacker configures Evilginx2 as a reverse proxy for Microsoft's login infrastructure:
- Proxy domain: A convincing look-alike (e.g., `login-microsoftonline.com`, `portal-azure-auth.com`)
- Target: `login.microsoftonline.com`
- Evilginx generates a **lure URL** — a short, innocent-looking link that routes through the proxy

**Lure URL format:**
```
https://suspicious-domain.com/5m4re3
```

### Phase 2: Phishing Delivery

The attacker crafts a phishing email containing the lure URL. Common pretexts:
- "Your account requires verification"
- "Document shared with you via OneDrive"
- "IT security policy update — action required"
- "Conference invitation — register here"

The email is sent to the target. Star Blizzard specifically targets:
- Government officials and diplomats
- Academic researchers (defense, policy)
- NGO staff in NATO member states
- Think tank analysts

### Phase 3: Credential and MFA Interception

When the victim clicks the lure:

1. **Victim sees**: A perfect replica of the Microsoft login page (it IS the real page, just proxied).
2. **Victim enters**: Username + password → forwarded to real Azure.
3. **Azure responds**: MFA challenge → displayed to victim through the proxy.
4. **Victim completes**: TOTP code or push notification → forwarded to Azure.
5. **Azure issues**: Authenticated session cookies → **intercepted by the proxy**.

The victim completes a legitimate login — they may not notice anything unusual.

### Phase 4: Session Hijacking

The attacker now has the authenticated session cookies:

| Cookie | Purpose |
|--------|---------|
| `ESTSAUTH` | Primary Azure AD session token |
| `ESTSAUTHPERSISTENT` | Persistent (long-lived) session token |

**To use the stolen cookies:**

1. Open a browser and navigate to `portal.azure.com`.
2. Open DevTools → Application → Cookies.
3. Manually set the `ESTSAUTH` and `ESTSAUTHPERSISTENT` cookies.
4. Refresh the page.
5. **You are now logged in as the victim** — no password or MFA needed.

### Phase 5: Post-Authentication Exploitation

Once inside the Azure Portal as the victim:

```
Check user properties → roles, group memberships
  → Access Azure resources (VMs, storage, databases)
    → Read emails via Outlook/Exchange
      → Access SharePoint/OneDrive files
        → Enumerate other users and escalate
```

Key post-exploitation actions:
- **Enumerate IAM roles**: What can this user access?
- **Check mailbox**: Search for sensitive conversations, credentials shared via email.
- **Access cloud resources**: VMs, databases, storage accounts.
- **Persistence**: Add forwarding rules, register new MFA device, create app registrations.
- **Lateral movement**: Use the compromised account to phish other users internally.

## Key Takeaways

### Why MFA doesn't stop this
MFA verifies that the person logging in is who they claim to be — but it doesn't protect the **session token** that is issued after successful authentication. AiTM attacks target the session, not the authentication process itself.

### Why it's hard to detect
- The victim completes a **real** login to the **real** Microsoft service.
- The proxy is transparent — all traffic passes through to the legitimate server.
- No malware is installed. No credentials are stored on disk.
- The only anomaly: the login IP comes from the attacker's proxy, not the victim's location.

### What stops AiTM attacks

| Defense | How it helps |
|---------|-------------|
| **FIDO2/WebAuthn hardware keys** | The key validates the real domain — refuses to authenticate on the proxy domain |
| **Conditional Access policies** | Block logins from unknown locations/IPs |
| **Token binding** | Ties the session token to the original TLS session |
| **Phishing-resistant MFA** | Hardware tokens that verify origin (YubiKey, etc.) |
| **Continuous access evaluation** | Revoke tokens when risk signals change |
| **User security awareness** | Train users to verify URLs before entering credentials |

### What does NOT stop AiTM
- SMS-based MFA ❌
- TOTP apps (Google Authenticator, Microsoft Authenticator push) ❌
- Email-based OTP ❌

All of these can be proxied through the AiTM framework.

## Tools Used in AiTM Attacks

| Tool | Description |
|------|-------------|
| **Evilginx2** | The most popular AiTM phishing framework — creates transparent reverse proxies |
| **Modlishka** | Similar to Evilginx, real-time HTTP reverse proxy for credential interception |
| **Muraena** | Automated reverse proxy for phishing |
| **Nekto** | Newer AiTM tool with improved evasion |

## APT Context: Star Blizzard

- **Attribution**: Russia (FSB-linked)
- **Active since**: 2017+
- **Targets**: UK/US government, NATO, academia, defense think tanks
- **Primary technique**: Spearphishing with AiTM for O365/Azure access
- **Goal**: Espionage — access to policy documents, defense communications, diplomatic cables
- **Notable**: NCSC and Microsoft jointly exposed their operations in 2023

## References

- `techniques/spring-boot-actuator-exploitation.md` (related infrastructure exploitation)
- `techniques/password-reset-abuse.md` (authentication attack techniques)
- `methodology/bug-bounty-playbook.md`
- [Microsoft Threat Intelligence: Star Blizzard](https://www.microsoft.com/en-us/security/blog/tag/star-blizzard/)
