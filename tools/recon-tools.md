# Recon Tools

## Overview
This page summarizes the main tools you use for reconnaissance and how they fit into your workflows. It is meant as a quick reference and as a teaching resource for readers following your methodology.

## Core Tools

### whatweb
- **Purpose**: Quickly fingerprint web technologies and server headers.
- **Example Commands**:
  ```bash
  whatweb https://target.com
  ```
- **Usage Tips**:
  - Run early to get a feel for the stack (CMS, frameworks, load balancers).
  - Save output alongside `nmap` and `httpx` results for correlation.

### nmap
- **Purpose**: Network service enumeration and basic web path discovery.
- **Example Commands**:
  ```bash
  nmap -sV -p 80,443 target.com --script http-enum
  ```
- **Usage Tips**:
  - Focus on a small set of relevant ports for web targets (80, 443, 8080, 8443, etc.).
  - Combine with HTTP scripts to identify hidden paths on unusual ports.

### ffuf
- **Purpose**: High-speed fuzzing for directories, files, and API endpoints.
- **Example Commands**:
  ```bash
  ffuf -w /usr/share/wordlists/dirb/big.txt -u https://target.com/FUZZ -o main_dirs.json
  ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
       -u https://target.com/FUZZ -o api_dirs.json
  ```
- **Usage Tips**:
  - Use specific lists for admin panels, APIs, and technology-specific paths.
  - Save output in JSON and grep for interesting status codes and sizes.

### gobuster
- **Purpose**: Directory and file brute forcing with a straightforward interface.
- **Example Commands**:
  ```bash
  gobuster dir -u https://target.com -w /usr/share/wordlists/dirb/big.txt -o gobuster_main.txt
  ```
- **Usage Tips**:
  - Use as a second opinion to ffuf with different wordlists.
  - Good for quickly checking a handful of hosts discovered from subdomains.

### subfinder (or equivalent)
- **Purpose**: Subdomain enumeration using multiple data sources.
- **Example Commands**:
  ```bash
  subfinder -d target.com -o subs_raw.txt
  ```
- **Usage Tips**:
  - Always follow with `httpx` to see what is actually alive.
  - Feed discovered hosts into screenshot tools (`gowitness`, `aquatone`).

### httpx
- **Purpose**: Probing hosts and URLs for HTTP(S) responses, titles, and technologies.
- **Example Commands**:
  ```bash
  httpx -l subs_raw.txt -status-code -title -tech-detect -o live_subs.txt
  httpx -l wayback_urls.txt -o accessible_old.txt
  ```
- **Usage Tips**:
  - Use `-status-code -title -tech-detect` to quickly understand the landscape.
  - Reuse `live_subs.txt` as input for directory brute forcing and screenshots.

### wpscan / droopescan / joomscan
- **Purpose**: CMS-specific recon and vulnerability discovery.
- **Example Commands**:
  ```bash
  wpscan --url https://target.com --enumerate p,t,tt,u
  droopescan scan drupal -u https://target.com
  joomscan -u https://target.com
  ```
- **Usage Tips**:
  - Run only after you know the target actually uses the relevant CMS.
  - Focus on enumeration output first, then pivot into manual testing.

### gowitness / aquatone
- **Purpose**: Taking and organizing screenshots of many hosts and paths.
- **Example Commands**:
  ```bash
  gowitness file -f live_subs.txt
  aquatone -ports xlarge < live_subs.txt
  ```
- **Usage Tips**:
  - Use early to visually prioritize interesting surfaces.
  - Combine with URL lists from `httpx`, `waybackurls`, or JS recon.

### arjun
- **Purpose**: Parameter discovery for endpoints.
- **Example Commands**:
  ```bash
  arjun -u https://target.com/login -o login_params.txt
  arjun -u https://target.com/search -m POST -o search_params.txt
  ```
- **Usage Tips**:
  - Run on login, search, and API endpoints to reveal hidden parameters.
  - Combine parameters with payload wordlists for injection testing.

### waybackurls / gau
- **Purpose**: Historical URL and JS discovery from archives.
- **Example Commands**:
  ```bash
  waybackurls target.com > wayback_urls.txt
  gau target.com > gau_urls.txt
  ```
- **Usage Tips**:
  - Filter for `.js`, `.php`, `.aspx`, etc. to find older entry points.
  - Feed into `httpx` to see which ones are still reachable.

### WaerJScan
- **Purpose**: Advanced JS-focused recon combining crawling, JS parsing, and wordlist generation.
- **High-level Workflow**:
  - Crawl the target domain.
  - Extract endpoints, parameters, emails, API keys, and “interesting strings”.
  - Generate a custom wordlist using TF-IDF and n-grams.
- **Usage Tips**:
  - Use for deep analysis on one high-value target at a time.
  - Reuse its `custom_wordlist.txt` and `endpoints.txt` in `ffuf`, Burp, and manual testing.

## Related Techniques

- `methodology/web-recon-methodology.md`
- `methodology/javascript-endpoint-discovery.md`
- `techniques/mass-assignment-vulnerabilities.md`
