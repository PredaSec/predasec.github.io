---
title: "Domain Enumeration using PowerView"
date: 2025-03-27 10:13:00 +0000
categories: [Certifications Technical Notes, Certified Red Team Professional (CRTP)]
tags: [active directory attacks, offensive security]
---

| **Technique** | **Command** | **Description** |
| --- | --- | --- |
| **List All Users** | `Get-DomainUser | select cn, logoncount` | Enumerate all domain users with logon counts |
| **List All Computers** | `Get-DomainComputer | select cn, logoncount` | Enumerate all domain computers with logon counts |
| **Get Domain Policy** | `Get-DomainPolicy` | Retrieve domain-wide policies |
| **Get Domain Controller Info** | `Get-DomainController` | List domain controllers and their details |
| **Get User Properties** | `Get-DomainUser -Identity studentx -Properties *` | Retrieve all properties for a specific user |
| **Get User's Groups** | `Get-DomainGroup -UserName 'studentx' | select cn` | List all groups a user belongs to |
| **Get Admin Group Info** | `Get-DomainGroup -Identity "Domain Admins"` | Retrieve information about Domain Admins group |
| **Get Local Groups** | `Get-NetLocalGroup -ComputerName dcorp-dc` | Enumerate local groups on a target computer |
| **Get Domain Groups** | `Get-NetGroup | select cn` | List all domain groups |
| **Get Group Members** | `Get-DomainGroupMember -Identity "Domain Admins" | select MemberName` | List members of a specific group |
| **List All OUs** | `Get-DomainOU | select name` | Enumerate all Organizational Units |
| **List Computers in OU** | `(Get-DomainOU -Identity DevOps).distinguishedname | %{Get-DomainComputer -SearchBase $_} | select dnshostname` | Find all computers within a specific OU |
| **List All GPOs** | `Get-DomainGPO | select name` | Enumerate all Group Policy Objects |
| **Get Local Administrators** | `Get-NetLocalGroup -ComputerName "dcorp-std486" -GroupName "Administrators"` | List local administrators on a target machine |
| **Get Sessions (OPSEC)** | `C:\AD\Tools\Invoke-SessionHunter.ps1; Invoke-SessionHunter -NoPortScan -RawResults -Targets .\computers.txt | select HostName,Access,UserSession` | Hunt for user sessions across multiple targets (OPSEC-safe) |
| **Find PSRemoting Admin Access** | `. C:\AD\Tools\Find-PSRemotingLocalAdminAccess.ps1; Find-PSRemotingLocalAdminAccess` | Discover machines with PowerShell Remoting admin access |
| **Find Domain Admin Sessions** | `Find-DomainUserLocation` | Locate computers where domain admins have active sessions |
| **Find Kerberoastable Users** | `Get-DomainUser -SPN` | Identify users with Service Principal Names (vulnerable to Kerberoasting) |
