# Waer's Cybersecurity Knowledge Base

> **50+ documents** · **20 technique deep dives** · **10 attack workflows** · **8 real-world case studies** · **Built from the field, not the textbook.**

---

## 👤 Who Am I

I'm **Waer** (Abdelrahman) — a cybersecurity researcher and bug bounty hunter focused on **web application security**, **cloud exploitation**, and **security automation**.

This repository is my brain, exported. Everything here comes from hands-on experience: breaking into bug bounty targets, solving CTF challenges, completing APT lab simulations, and building my own security tooling. I don't write about attacks I've only read about — I write about attacks I've executed, debugged, and reported.

**Areas I work in:**
- 🎯 **Bug Bounty Hunting** — web app recon, vulnerability discovery, and responsible disclosure
- 🔐 **Penetration Testing** — web, API, cloud infrastructure, and CMS-specific testing
- 🏴 **CTF Competitions** — web exploitation, forensics, steganography, and crypto
- 🤖 **Security Automation** — custom recon tools, JS crawlers, and AI-powered security agents
- ☁️ **Cloud Security** — AWS exploitation, Spring Boot misconfigurations, metadata attacks
- 🕵️ **APT Emulation** — simulating real-world threat actors (Star Blizzard, Volkswagen breach)

📬 **Get in touch:**
- 📧 Email: [abdowaer099@gmail.com](mailto:abdowaer099@gmail.com)
- 💼 LinkedIn: [linkedin.com/in/wa3r](https://linkedin.com/in/wa3r)

---

## 🔥 What's Inside

This isn't a wiki dump or a copy-paste of OWASP pages. Every document here follows a battle-tested structure: **detect it → exploit it → escalate it → report it**, with real payloads, real tool commands, and real decision trees.

| Section | Count | What you'll find |
|---------|-------|-----------------|
| **[methodology/](methodology/)** | 5 docs | Full workflows from recon to exploitation to privilege escalation |
| **[techniques/](techniques/)** | 20 docs | Deep dives into every major web vulnerability class with working payloads |
| **[scenarios/](scenarios/)** | 10 docs | Step-by-step attack playbooks you can follow during live testing |
| **[case-studies/](case-studies/)** | 8 docs | Real findings and APT simulations — anonymized but fully detailed |
| **[web-vulnerabilities/](web-vulnerabilities/)** | 6 docs | Index pages grouping vulns by category with cross-references |
| **[tools/](tools/)** | 4 docs | Tool catalogs, 90+ Google dorks, curated payload lists |
| **[forensics/](forensics/)** | 3 docs | Investigation workflows, image forensics, steganography detection |

**Vulnerability coverage:** SQL Injection · XSS · SSRF · SSTI · XXE · Command Injection · IDOR · File Upload · JWT Attacks · CORS · Cache Poisoning · Prototype Pollution · Race Conditions · Mass Assignment · Open Redirect · Password Reset Abuse · Deserialization RCE · WordPress Hacking · Spring Boot Actuator · AiTM MFA Bypass

---

## 🎯 Who This Is For

### 🏴‍☠️ Bug Bounty Hunters
You'll find ready-to-use **attack playbooks** with exact payloads, WAF bypass techniques, and vulnerability chaining patterns (XSS + CORS = ATO, Open Redirect + OAuth = token theft). The methodology docs cover the full lifecycle: target selection → recon → exploitation → escalation → reporting. No fluff — just what works.

### 🔓 Penetration Testers
The **privilege escalation checklist** covers web, Linux, Windows, and AWS in one document. Technique docs include detection commands, exploitation steps, and filter bypass tables you can reference mid-engagement. Scenarios serve as step-by-step checklists during assessments.

### 🏁 CTF Players
Case studies include **CTF write-ups** for SSTI, XXE, DNS zone transfers, and S3 bucket misconfigurations. The forensics section covers JPEG/PNG steganography workflows, and the tools section has the exact analysis order for forensics challenges. The SSTI workflow includes a **decision tree** for fingerprinting template engines.

### 🧪 CTF Challenge Makers
Understand how attackers approach your challenges. The scenario docs reveal the **thought process and decision trees** players use — from initial detection probes to full exploitation chains. Use this to design better, more realistic challenges.

### 📚 Security Students & Researchers
Every technique doc is structured as a **learning path**: what the vulnerability is → when it happens → how to find it → how to exploit it → how to escalate impact → what tools to use. Start with methodology, then go deep into any technique that interests you.

### 🛡️ Blue Team / Defenders
Each technique doc includes **what to look for** from the attacker's perspective. Understanding how attackers chain vulnerabilities helps you build better detections, write better rules, and prioritize hardening efforts.

---

## 📂 Repository Structure

### 📋 [methodology/](methodology/)
Processes and playbooks from recon to privilege escalation:
- [Web Recon Methodology](methodology/web-recon-methodology.md) — 11-phase structured recon pipeline
- [JavaScript Endpoint Discovery](methodology/javascript-endpoint-discovery.md) — JS-focused recon with automated crawling
- [Bug Bounty Playbook](methodology/bug-bounty-playbook.md) — Program selection, recon-to-exploit flow, reporting tips
- [Exploitation Methodology](methodology/exploitation-methodology.md) — Systematic exploitation, impact escalation, and vulnerability chaining
- [Privilege Escalation Checklist](methodology/privilege-escalation-checklist.md) — Web, Linux, Windows, and AWS cloud privesc

### 🎯 [techniques/](techniques/)
20 technique-focused deep dives with working payloads:
- [SQL Injection](techniques/sql-injection.md) — UNION, blind, time-based, OOB, WAF bypasses, SQLi→RCE
- [XSS Techniques and Payloads](techniques/xss-techniques.md) — Context-aware payloads, filter bypasses, WAF evasion
- [Command Injection](techniques/command-injection.md) — Operators, reverse shells, filter bypass techniques
- [SSRF (Server-Side Request Forgery)](techniques/ssrf-server-side-request-forgery.md) — Cloud metadata, protocol smuggling, DNS rebinding
- [Server-Side Template Injection](techniques/server-side-template-injection.md) — Engine fingerprinting, Jinja2/Twig/Freemarker RCE
- [XXE Injection](techniques/xxe-injection.md) — File read, SSRF, blind/OOB extraction
- [IDOR (Insecure Direct Object Reference)](techniques/idor-insecure-direct-object-reference.md) — Horizontal/vertical escalation, UUID techniques
- [JWT Attacks and Misconfigurations](techniques/jwt-attacks-and-misconfigurations.md) — Algorithm confusion, key injection, brute force
- [File Upload Vulnerabilities](techniques/file-upload-vulnerabilities.md) — Extension/magic byte bypasses, webshells, SVG attacks
- [Race Conditions](techniques/race-conditions.md) — Turbo Intruder, HTTP/2 single-packet, financial exploitation
- [CORS Misconfigurations](techniques/cors-misconfigurations.md) — Origin reflection, null origin, subdomain trust abuse
- [Web Cache Poisoning](techniques/web-cache-poisoning.md) — Unkeyed headers, cache deception
- [Password Reset Abuse](techniques/password-reset-abuse.md) — Token leakage, host header injection, email parameter pollution
- [Mass Assignment Vulnerabilities](techniques/mass-assignment-vulnerabilities.md) — Role injection, price manipulation
- [Client-Side Prototype Pollution](techniques/prototype-pollution-client-side.md) — Source-gadget model, DOM XSS via PP
- [Server-Side Parameter Pollution](techniques/prototype-pollution-server-side.md) — Query string and REST path injection
- [JSON Deserialization RCE](techniques/json-deserialization-rce.md) — Java, Python, Node, PHP, .NET gadget chains
- [WordPress Hacking Methodology](techniques/wordpress-hacking.md) — XML-RPC, REST API, plugin/theme exploitation
- [Spring Boot Actuator Exploitation](techniques/spring-boot-actuator-exploitation.md) — Heap dump, env, gateway route injection
- [Open Redirect](techniques/open-redirect.md) — Filter bypasses, OAuth/phishing/SSRF chaining

### 🔄 [scenarios/](scenarios/)
10 step-by-step attack workflows with decision trees:
- [SQL Injection Testing](scenarios/sql-injection-testing-workflow.md)
- [File Upload to RCE](scenarios/file-upload-to-rce-workflow.md)
- [IDOR Discovery and Escalation](scenarios/idor-discovery-and-escalation-workflow.md)
- [SSTI Detection and Exploitation](scenarios/ssti-detection-and-exploitation-workflow.md)
- [JWT Authentication Bypass](scenarios/jwt-authentication-bypass-workflow.md)
- [Race Condition Exploitation](scenarios/race-condition-exploitation-workflow.md)
- [Reflected XSS Testing](scenarios/reflected-xss-testing-workflow.md)
- [SSRF via XML-RPC Pingback](scenarios/ssrf-via-xmlrpc-pingback-workflow.md)
- [Logic Flaw: Unauthorized Checkout](scenarios/logic-flaw-unauthorized-checkout-workflow.md)
- [Spring Boot Actuator to Cloud Compromise](scenarios/spring-boot-actuator-to-cloud-compromise-workflow.md)

### 📖 [case-studies/](case-studies/)
Real findings and APT simulations — fully anonymized:
- [SSRF via WordPress XML-RPC](case-studies/case-study-ssrf-xmlrpc-wordpress.md) — Real bug bounty finding
- [Checkout Password Leak](case-studies/case-study-checkout-password-leak.md) — Client-side auth bypass
- [WordPress REST API Enumeration](case-studies/case-study-recon-wordpress-api-enumeration.md) — API surface mapping
- [SSTI to Database Access (CTF)](case-studies/case-study-ssti-rce-ctf.md) — Flask/Jinja2 exploitation
- [XXE File Read (CTF)](case-studies/case-study-xxe-ctf.md) — Firmware update XML injection
- [DNS Zone Transfer & S3 Bucket (CTF)](case-studies/case-study-dns-zone-transfer-s3-ctf.md) — Infrastructure misconfig
- [Spring Boot Actuator — VW Breach Simulation](case-studies/case-study-spring-boot-actuator-volkswagen.md) — Heap dump → AWS → Secrets Manager
- [MFA Bypass via AiTM — Star Blizzard APT](case-studies/case-study-aitm-mfa-bypass-star-blizzard.md) — Russian APT session cookie theft

### 🌐 [web-vulnerabilities/](web-vulnerabilities/)
Index pages grouping related content:
- [XSS](web-vulnerabilities/xss.md) · [SSRF](web-vulnerabilities/ssrf.md) · [Auth & Session Issues](web-vulnerabilities/auth-and-session-issues.md)
- [Injection](web-vulnerabilities/injection.md) · [Access Control & Logic](web-vulnerabilities/access-control-and-business-logic.md) · [File Upload & Deserialization](web-vulnerabilities/file-upload-and-deserialization.md)

### 🛠️ [tools/](tools/)
- [Recon Tools](tools/recon-tools.md) · [Forensics Tools](tools/forensics-tools.md) · [Wordlists & Payloads](tools/wordlists-and-payload-lists.md) · [Google Dorking (90+ dorks)](tools/google-dorking.md)

### 🔍 [forensics/](forensics/)
- [Forensics Workflow](forensics/forensics-workflow.md) · [JPEG Forensics](forensics/image-forensics-jpeg.md) · [PNG Forensics](forensics/image-forensics-png.md)

---

## 🗺️ How to Navigate

### If you're new here
```
Start here ──→ methodology/web-recon-methodology.md
          ──→ methodology/bug-bounty-playbook.md
          ──→ Pick any technique ──→ Read matching scenario ──→ Study the case study
```

### If you're mid-assessment
```
Find the vuln type ──→ techniques/{vuln}.md (detection + payloads)
Follow the workflow ──→ scenarios/{vuln}-workflow.md (step-by-step)
Copy payloads from ──→ tools/wordlists-and-payload-lists.md
```

### If you're solving a CTF
```
Web challenge ──→ techniques/ + scenarios/ (exploitation)
Forensics    ──→ forensics/forensics-workflow.md (analysis order)
Stego        ──→ forensics/image-forensics-*.md (JPEG/PNG specific)
Cloud        ──→ case-studies/ (AWS, Spring Boot, metadata)
```

---

## ⚠️ Disclaimer

All case studies are **fully anonymized** — real target domains, IPs, credentials, and personally identifiable information have been removed or replaced with placeholders. This repository is intended for **educational purposes** and **ethical security research** only. Always obtain proper authorization before testing any system.

## 🤝 Contributing

Contributions are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding new techniques, case studies, or improving existing content.

## 📄 License

This project is licensed under the **MIT License** — see [LICENSE](LICENSE) for details.

---

<p align="center">
  <b>Built by <a href="https://linkedin.com/in/wa3r">Waer</a></b> · 
  <a href="mailto:abdowaer099@gmail.com">abdowaer099@gmail.com</a> · 
  <a href="https://linkedin.com/in/wa3r">LinkedIn</a>
</p>
