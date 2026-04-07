---
title: "Shares Enumeration using PowerHuntShares"
date: 2025-03-27 10:12:00 +0000
categories: [Certifications Technical Notes, Certified Red Team Professional (CRTP)]
tags: [active directory attacks, offensive security]
---

- Import `PowerHuntShares` module
    
    ```powershell
    Import-Module C:\AD\Tools\PowerHuntShares.psm1
    ```
    
- Execute share hunt on multiple hosts (excluding DCs for OPSEC)
    
    ```powershell
    Invoke-HuntSMBShares -NoPing -OutputDirectory C:\AD\Tools\ -HostList C:\AD\Tools\servers.txt
    ```
