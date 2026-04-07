---
title: "Forest/Trust Enumeration with PowerView"
date: 2025-03-27 10:09:00 +0000
categories: [Certifications Technical Notes, Certified Red Team Professional (CRTP)]
tags: [active directory attacks, offensive security]
---

| **Technique** | **Command** | **Description** |
| --- | --- | --- |
| **Enumerate Forest** | `Get-ForestDomain -Verbose` | List all domains in the current forest |
| **Map Current Domain Trusts** | `Get-DomainTrust` | Enumerate trust relationships for the current domain |
| **Map Forest External Trusts** | `Get-ForestDomain` | Discover external trusts for the entire forest |
| **Map Domain External Trusts** | `Get-DomainTrust` | Identify external trust relationships for a specific domain |
| **Enumerate All Forest Domain Trusts** | `Get-ForestDomain -Forest eurocorp.local` | List all domains and their trusts within a specified forest |
