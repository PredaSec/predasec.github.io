---
title: "Enumerate ACLs using PowerView"
date: 2025-03-27 10:10:00 +0000
categories: [Certifications Technical Notes, Certified Red Team Professional (CRTP)]
tags: [active directory attacks, offensive security]
---

| **Technique** | **Command** | **Description** |
| --- | --- | --- |
| **Enumerate Domain Admins ACLs** | `Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs -Verbose` | List all ACL entries for the Domain Admins group with resolved GUIDs |
| **Check User Permissions (studentx)** | `Find-InterestingDomainAcl -ResolveGUIDs` | Discover interesting ACLs and permissions for specific user (e.g., studentx) |
| **Check RDP Group Permissions** | `Find-InterestingDomainAcl -ResolveGUIDs | Where-Object {$_.ObjectDN -match "Remote Desktop Users"}` | Identify notable ACL entries for RDP group members |
