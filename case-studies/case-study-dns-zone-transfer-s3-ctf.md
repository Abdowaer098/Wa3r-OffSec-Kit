# Case Study: DNS Zone Transfer and S3 Bucket Misconfiguration (CTF)

## Overview
This case study documents two related CTF challenges: exploiting a DNS zone transfer to extract flags, and leveraging S3 bucket permission misconfigurations to access private files. Both demonstrate common infrastructure misconfigurations that are frequently found in real-world environments.

## Challenge 1: DNS Zone Transfer

### Objective
Perform a zone transfer on a subdomain to retrieve the flag.

### Approach

**Step 1: Identify authoritative nameservers**
```bash
dig target-subdomain.example.com NS +short
```

The query revealed that the subdomain was its **own authoritative nameserver** — meaning DNS was self-hosted on the same domain.

**Step 2: Attempt zone transfer (AXFR)**
```bash
dig axfr @target-subdomain.example.com target-subdomain.example.com
```

The zone transfer succeeded because:
- The DNS server allowed AXFR from any source IP.
- No IP-based access controls were configured.

**Step 3: Analyze results**
The zone transfer returned the complete DNS zone data, including:
- The flag (stored as a TXT record).
- Additional nameserver entries (`ns1.*`).
- Internal hostnames not visible through normal DNS queries.

### Key Insight
Zone transfers should be restricted to authorized secondary DNS servers only. When unrestricted, they expose the **entire DNS zone** — including internal hostnames, service records, and in CTF contexts, flags hidden in TXT records.

---

## Challenge 2: Internal Zone Transfer

### Objective
Perform a zone transfer on an internal DNS zone using a specific nameserver.

### Approach

```bash
dig axfr int @ns.target.example.com
```

Here, `int` is the internal zone name, and the nameserver is specified with `@`. The transfer succeeded because:
- The internal zone was queryable from external networks.
- AXFR was unrestricted.

This revealed all internal zone records, including the flag.

### Key Insight
Internal DNS zones should **never** be accessible from external networks. Two failures here:
1. The nameserver for the internal zone was reachable from the internet.
2. AXFR was not restricted to authorized servers.

---

## Challenge 3: AWS S3 Bucket Permission Bypass

### Objective
Access a file (`key2.txt`) on an S3 bucket that requires authentication.

### Background: "Authenticated Users" Misconfiguration
AWS S3 formerly had an "Any Authenticated Users" permission category. Many administrators mistakenly believed this meant "any user in my AWS organization." In reality, it meant **any AWS account holder worldwide**.

### Approach

**Step 1: Try unauthenticated access**
```bash
aws s3 ls s3://target-bucket --no-sign-request
```
This failed — confirming the bucket requires authentication.

**Step 2: Authenticate with any AWS account**
```bash
# Configure AWS CLI with any valid credentials
aws configure
# Enter: Access Key ID, Secret Key, Region
```

**Step 3: Access the bucket**
```bash
aws s3 ls s3://target-bucket
aws s3 ls s3://target-bucket --recursive | grep key2.txt
```

**Step 4: Download the target file**
```bash
aws s3 cp s3://target-bucket/key2.txt ./key2.txt
```

The bucket was accessible because the owner used "Any Authenticated Users" permission, which grants access to the entire AWS user base.

### Key Insight
The "Authenticated Users" permission in AWS means **any valid AWS account**, not just your organization. This is one of the most common S3 misconfiguration patterns. AWS has since deprecated this option, but legacy buckets may still have it.

## Lessons Learned

1. **DNS zone transfers** should be restricted by IP to authorized secondary servers only. Use `allow-transfer { trusted-ips; };` in BIND configuration.
2. **Internal DNS zones** must not be accessible from external networks — segment DNS infrastructure.
3. **S3 bucket permissions** are confusing by design — "Authenticated Users" does NOT mean "my team." Always use explicit IAM policies.
4. **The `dig axfr` command** is the standard test for zone transfer vulnerabilities — include it in every external recon checklist.
5. **Cloud storage enumeration** should be part of every bug bounty recon workflow — use `cloud_enum`, `s3scanner`, or manual `aws s3 ls`.

## References

- `methodology/web-recon-methodology.md` (Phase 17: DNS, Phase 20: Cloud Storage)
- `tools/recon-tools.md`
- `methodology/bug-bounty-playbook.md`
