# Case Study: WordPress REST API Enumeration and Risk Assessment

## Overview
This case study documents a thorough reconnaissance of a WordPress site's REST API (`wp-json`), revealing the full surface of API endpoints, their HTTP methods, authentication requirements, and the sensitivity of exposed data. The enumeration produced a risk-rated table that guided further vulnerability testing.

## Target Context (anonymized)

- **Platform**: WordPress 6.x with WooCommerce, MailPoet, Jetpack, Astra theme.
- **Database**: MariaDB 11.x.
- **Infrastructure**: Shared hosting with exposed database port (3306) and SMTP relay.
- **Users discovered**: Multiple accounts via WPScan enumeration (admin, content editors, order managers).

## Vulnerability

**Type**: Information Disclosure / Excessive API Exposure

The WordPress REST API (`/wp-json/wp/v2/`) was largely accessible without authentication, and even authenticated endpoints revealed their existence and expected parameters. Combined with user enumeration and technology fingerprinting, this gave a complete map of the attack surface.

## Recon and Discovery

### 1. Technology fingerprinting
```bash
whatweb https://target.com
nmap -sV -p 80,443,3306,587 target.com
```

Results revealed:
- WordPress version
- MariaDB version and exposed port
- SMTP service on port 587

### 2. User enumeration
```bash
wpscan --url https://target.com --enumerate u
```
Discovered multiple user accounts with sequential IDs.

### 3. API endpoint mapping

Accessed `/wp-json/wp/v2/` to get the full API index, then systematically tested each endpoint:

| Endpoint | Methods | Description | Sensitive Information | Risk Level | Requires Auth |
|----------|---------|-------------|----------------------|------------|---------------|
| `/wp/v2/users` | GET, POST | List/create users | Usernames, emails, IDs, roles | **HIGH** | Yes |
| `/wp/v2/users/{id}` | GET, POST, PUT, PATCH, DELETE | Individual user operations | User details, metadata, roles | **HIGH** | Yes |
| `/wp/v2/feedback` | GET, POST, PUT, PATCH, DELETE | Jetpack feedback forms | Contact form submissions, user data | **HIGH** | Yes |
| `/wp/v2/mailpoet_email` | GET, POST | MailPoet email marketing | Email templates, subscriber data | **HIGH** | Yes |
| `/wp/v2/jp_pay_order` | GET, POST | Jetpack Pay orders | Payment information, customer data | **HIGH** | Yes |
| `/wp/v2/jp_pay_product` | GET, POST | Jetpack Pay products | Product pricing, payment settings | **HIGH** | Yes |
| `/wp/v2/plugins` | GET, POST | Plugin management | Installed plugins, versions | **HIGH** | Yes |
| `/wp/v2/settings` | GET, POST | Site settings | Site configuration, admin settings | **HIGH** | Yes |
| `/wp/v2/courses` | GET, POST | Course content (LMS) | Course materials, student data | **MEDIUM** | Yes |
| `/wp/v2/comments` | GET, POST | Comments system | Comment content, author emails/IPs | **MEDIUM** | Yes |
| `/wp/v2/posts` | GET, POST | Blog posts | Published/draft posts, author IDs | **MEDIUM** | Yes |
| `/wp/v2/pages` | GET, POST | Static pages | Page content, author information | **MEDIUM** | Yes |
| `/wp/v2/themes` | GET | Theme management | Active/installed themes, versions | **MEDIUM** | Possibly No |
| `/wp/v2/product` | GET, POST | WooCommerce products | Product data, pricing, inventory | **MEDIUM** | Yes |
| `/wp/v2/users/me` | GET, POST, PUT, PATCH, DELETE | Current user operations | Current authenticated user info | **MEDIUM** | Yes |
| `/wp/v2/media` | GET, POST | Media library | Uploaded files, metadata | **LOW** | Yes |
| `/wp/v2/categories` | GET, POST | Post categories | Content organization | **LOW** | Yes |
| `/wp/v2/tags` | GET, POST | Post tags | Content tags and metadata | **LOW** | Yes |
| `/wp/v2` | GET | Main API endpoint | All available endpoints | **INFO** | Possibly No |

### 4. Key findings from API probing

- **Media endpoint** (`/wp/v2/media?parent=`) returned a large number of results when queried with different parent IDs, indicating many uploaded files with accessible metadata.
- **Author ID enumeration**: The `/wp/v2/users/` endpoint combined with author archives (e.g., `/author/{username}/`) revealed user IDs and mappings.
- **Plugin and theme exposure**: Even without authenticated access, the existence and types of endpoints (MailPoet, Jetpack Pay, WooCommerce) revealed the exact plugin stack.

## Impact

| Impact area | Description |
|------------|-------------|
| **User enumeration** | Usernames and IDs exposed via API and archives |
| **Technology disclosure** | Exact versions of CMS, database, plugins, and themes |
| **Attack surface mapping** | Full API surface documented with methods and data types |
| **Data exposure risk** | Payment, email, feedback, and user data accessible if auth is weak |
| **Further attack guidance** | Known plugin versions → targeted CVE search |

## Lessons Learned

1. **`/wp-json/` is the first stop** when testing any WordPress site — it maps the entire API surface in one request.
2. **Risk-rating API endpoints** (HIGH/MEDIUM/LOW) based on the data they handle helps prioritize testing and communicate findings clearly.
3. **Plugin-specific endpoints** (MailPoet, Jetpack, WooCommerce) inherit their own vulnerability history — always check for known CVEs against the discovered versions.
4. **Exposed database ports** (3306 from outside) and SMTP relays are infrastructure-level findings worth reporting alongside web-layer issues.
5. **User enumeration** in WordPress is almost always possible through multiple channels (API, author archives, login error messages) — the risk is in what you combine it with.

## References

- `methodology/web-recon-methodology.md`
- `tools/recon-tools.md`
- `case-studies/case-study-ssrf-xmlrpc-wordpress.md`
