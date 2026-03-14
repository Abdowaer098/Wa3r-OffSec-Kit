# Contributing to Waer's Cybersecurity Knowledge Base

Thanks for your interest in contributing! This repository is built from real-world security research, and contributions that follow the same standard are welcome.

## What You Can Contribute

- **New technique documents** — deep dives on vulnerability classes not yet covered
- **New scenario workflows** — step-by-step attack playbooks
- **New case studies** — anonymized write-ups of real findings or CTF solutions
- **Improvements** — fixing errors, adding missing payloads, updating tool references
- **Translations** — making content accessible in more languages

## Document Standards

### Technique documents (`techniques/`)
Every technique doc should follow this structure:
1. **Overview** — What the vulnerability is
2. **When This Happens** — Common conditions and attack surface
3. **Recon / Detection** — How to identify the vulnerability
4. **Exploitation Steps** — Numbered, reproducible steps with working payloads
5. **Tools** — Relevant tools with example commands
6. **Real-World Notes** — Practical tips from experience
7. **References** — Links to related docs in the repository

### Scenario workflows (`scenarios/`)
Scenarios should be step-by-step workflows with:
- Numbered phases and steps
- Exact commands and payloads
- A decision tree at the end
- Tool and technique references

### Case studies (`case-studies/`)
Case studies must be:
- **Fully anonymized** — no real domains, IPs, credentials, or PII
- Written as a narrative with clear steps
- Include lessons learned

## Rules

1. **No sensitive data** — Never include real target domains, IP addresses, credentials, API keys, cookies, or personally identifiable information. Use placeholders like `target.com`, `ATTACKER_IP`, `REDACTED`.
2. **No copy-paste from other sources** — Write original content based on your own understanding and experience. Reference external sources properly.
3. **English only** — All content should be in English for accessibility.
4. **Test your payloads** — Make sure commands and payloads are syntactically correct.

## How to Submit

1. Fork this repository
2. Create a feature branch (`git checkout -b add-new-technique`)
3. Add your content following the standards above
4. Update the relevant `README.md` files (section README + main README)
5. Submit a pull request with a clear description of what you added

## Questions?

Reach out:
- 📧 [abdowaer099@gmail.com](mailto:abdowaer099@gmail.com)
- 💼 [linkedin.com/in/wa3r](https://linkedin.com/in/wa3r)
