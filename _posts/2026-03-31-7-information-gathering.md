---
title: "Information Gathering"
date: 2026-03-31 10:14:00 +0000
categories: [Certifications Technical Notes, Certified Penetration Testing Specialist (CPTS)]
tags: [enumeration, offensive security, networking]
---


*“Enumeration is the art of identifying all of the ways we could attack a target”* 

## NMAP

### SYN Scan

- `sS` Scan :

    - In an SYN scan, Nmap sends a SYN packet to the target port:

        - If the target responds with a SYN-ACK, Nmap identifies the port as open.
        - If the target responds with an RST, the port is considered closed.
        - If no response is received, the port is marked as filtered, which typically indicates that a firewall is dropping or blocking the packets.

    - This technique is often referred to as a half-open scan because the TCP handshake is not fully completed.

    - Advantages:

        - Stealthier than full TCP connect scans (does not complete the handshake)
        - Faster execution
        - Low resource usage
        - Can bypass some firewall rules
        - Provides reliable detection of open ports   

### Host Discovery

- Scan network range (ICMP echo requests)
    
    ```bash
    sudo nmap 10.129.2.0/24 -sn -oA tnet | grep for | cut -d" " -f5
    # -sn disable port scan
    ```
    
- Scan multiple IPs
    
    ```bash
    sudo nmap -sn -oA tnet 10.129.2.18 10.129.2.19 10.129.2.20| grep for | cut -d" "
    sudo nmap -sn -oA tnet 10.129.2.18-20| grep for | cut -d" " -f5 -f5
    ```
    
- Scan single IP
    
    ```bash
    sudo nmap 10.129.2.18 -sn -oA host
    sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace #ensure ping scan and show all packets sent and received
    # --reason to show the reason
    # --disable-arp-ping => only ICMP echo requests
    ```
    
    ⇒ If we disable port scan (`-sn`), Nmap automatically ping scan with `ICMP Echo Requests` (`-PE`).<br>
    ⇒ we can know the operating system by checking it’s `ttl` value in the echo response message.
    

### Host and Port Scanning

- Port states :

    | `unfiltered` | This state of a port only occurs during the **TCP-ACK** scan and means that the port is accessible, but it cannot be determined whether it is open or closed. |
    | `open|filtered` | If we do not get a response for a specific port, `nmap` will set it to that state. This indicates that a firewall or packet filter may protect the port. |
    | `closed|filtered` | This state only occurs in the **IP ID idle** scans and indicates that it was impossible to determine if the scanned port is closed or filtered by a firewall. |

- Scanning top 10 TCP ports :
    
    ```bash
    sudo nmap 10.129.2.28 --top-ports=10 #21,22,23,25,80,110,139,443,445,3389
    # -F top 100 ports
    ```
    
- Trace the packets without ICMP and ARP ping :
    
    ```bash
    sudo nmap 10.129.2.28 -p 21 --packet-trace -Pn -n --disable-arp-ping
    # -n disable DNS resolution
    # -Pn disable ICMP echo request
    ```
    
- Connect scan : which is the `TCP` scan and it’s considered the most stealthy because it wont leave any unfinished connections or unsent packets (slower)
- UDP port scan : If we get an ICMP response with `error code 3` (port unreachable), we know that the port is indeed `closed`. For all other ICMP responses, the scanned ports are marked as (`open|filtered`).

### Saving Output

- We can easily save the output to be read in a `html` page using `xsltproc` command after extracting it as `xml` file with `-oX` or `-oA` , very helpful for documentation.
    
    ```bash
    xsltproc target.xml -o target.html
    ```
    

### Service Enumeration

- To define how periods of time the status should be shown we can use `--stats-every=5s/m` or increase verbosity with `-v` and `-vv` for every port found
- To grab the banner of some service with `nc` or `tcpdump` :
    1. Execute `tcpdump` to intercept traffic
        
        ```bash
        sudo tcpdump -i eth0 host 10.10.14.2 and 10.129.2.28
        ```
        
    2. Connect to the service using the given port
        
        ```bash
        nc -nv 10.129.2.28 25
        ```
        
    3. Now we can see the the three-way handshake providing all the required data

### Scripting Engine




| **Category** | **Description** |
| --- | --- |
| `auth` | Determination of authentication credentials. |
| `broadcast` | Scripts, which are used for host discovery by broadcasting and the discovered hosts, can be automatically added to the remaining scans. |
| `brute` | Executes scripts that try to log in to the respective service by brute-forcing with credentials. |
| `default` | Default scripts executed by using the `-sC` option. |
| `discovery` | Evaluation of accessible services. |
| `dos` | These scripts are used to check services for denial of service vulnerabilities and are used less as it harms the services. |
| `exploit` | This category of scripts tries to exploit known vulnerabilities for the scanned port. |
| `external` | Scripts that use external services for further processing. |
| `fuzzer` | This uses scripts to identify vulnerabilities and unexpected packet handling by sending different fields, which can take much time. |
| `intrusive` | Intrusive scripts that could negatively affect the target system. |
| `malware` | Checks if some malware infects the target system. |
| `safe` | Defensive scripts that do not perform intrusive and destructive access. |
| `version` | Extension for service detection. |
| `vuln` | Identification of specific vulnerabilities. |



- Scan for specific script category :
    
    ```bash
    sudo nmap <target> --script <category>
    sudo nmap <target> --script <script-name>,<script-name>,... # for defined scripts
    ```
    
    ⇒ with `-script banner` we can further enumerate an unknown service without the use of `nc` and `tcpdump` 
    
- Aggressive scan (`-A`) scans the target with multiple options as service detection (`-sV`), OS detection (`-O`), traceroute (`--traceroute`), and with the default NSE scripts (`-sC`).
- Vulnerability assessment scan :
    
    ```bash
    sudo nmap 10.129.2.28 -p 80 -sV --script vuln
    ```
    

### Performance

- **Timeouts**
    
    Optimized timeouts (`Round-Trip-Time` - `RTT`) for faster scan, but it may us to overlook hosts, default is `100ms`
    
    ```bash
    sudo nmap 10.129.2.0/24 -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms
    ```
    
- **Max retries**
    
    Reducing max retries with (`--max-retries`) will increase the speed of our scan but again it can overlook important information, default is `10`
    
    ```bash
    sudo nmap 10.129.2.0/24 -F --max-retries 0
    ```
    
- **Rates**
    
    During white-box penetration testing to assess the network we might know the network bandwidth , we can work with the rate of packets sent to speed up our scan without missing any information
    
    ```bash
    sudo nmap 10.129.2.0/24 -F -oN tnet.minrate300 --min-rate 300
    ```
    

### Firewall and IDS/IPS Evasion

- **ACK scan**
    
    With the `ACK` scan `-sA` ,`nmap` will only use `ACK` packets to determine port’s availability which will be much harder to filter by IDS/IPS systems
    
    ```bash
    sudo nmap 10.129.2.28 -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace
    ```
    
- **IDS/IPS detection**
    
    Scenario : trigger a certain host, service or port with an aggressive scan (`-A` and `T5` for example) with a VPS and based on whether specific security measures are taken, we can detect if the network has some monitoring applications or not.
    
- **Decoys**
    
    We can disguise our IP between fake IP@s using `-D RND:5` (which must be alive),where RND is the generate number and then we specify how many random numbers to generate. We use this in cases where administrators block specific sub-nets from different regions or IPS is primarily blocking us. 
    
    - Scan using decoys :
        
        ```bash
        sudo nmap 10.129.2.28 -p 80 -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5
        ```
        
    - Scan using different source IP :
        
        ```bash
        sudo nmap 10.129.2.28 -n -Pn -p 445 -O -S 10.129.2.200 -e tun0
        ```
        
- **DNS proxying**
    
    This technique can be used to exploit weaknesses in firewalls that are improperly configured to blindly accept incoming traffic based on a specific port number such as 53, 20 and 67
    
    ```bash
    sudo nmap 10.129.2.28 -p50000 -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53
    # after we have found that firewall accepts TCP port 53, we can test IDS/IPS by connection with netcat
    ncat -nv -p 53 10.129.2.28 50000 
    ```
    
- **Fragmentation (**the packet is split into 3 fragments)
    
    ```bash
    nmap -f 1.1.1.1 
    ```
    
- **Scan delay**
    
    This is very useful if the network is unstable ,but also for evading any time-based firewall/IDS triggers which may be in place (time in MS)
    
    ```bash
    nmap --scan-delay 10 192.168.1.13
    ```

## Network Services Footprinting

### FTP

- TFTP : UDP | No Authentication | Does not have directory listing functionality
    
    - TFTP commands    
        
        | **Commands** | **Description** |
        | --- | --- |
        | `connect` | Sets the remote host, and optionally the port, for file transfers. |
        | `get` | Transfers a file or set of files from the remote host to the local host. |
        | `put` | Transfers a file or set of files from the local host onto the remote host. |
        | `quit` | Exits tftp. |
        | `status` | Shows the current status of tftp, including the current transfer mode (ascii or binary), connection status, time-out value, and so on. |
        | `verbose` | Turns verbose mode, which displays additional information during file transfer, on or off. |

- `/etc/ftpusers` is used to deny certain users from FTP access
- to list `vsFTPd` server settings : `ftp> status`
- for more verbose responses from the server we can enable debug and trace mode via : `debug`, `trace`
- for recursive listing : `ls -R`
- Download all files :
    
    ```bash
    $ wget -m --no-passive ftp://anonymous:anonymous@10.129.29.31
    ```
    
- Enumerate `ftp` server that runs with `TLS/SSL` encryption :
    
    ```bash
    $ openssl s_client -connect 10.129.14.136:21 -starttls ftp
    ```
    

### SMB

- Samba implements the Common Internet File System (`CIFS`) network protocol
- SMBclient commands :
    
    ```bash
    # List shares
    smbclient -N -L //10.10.12.13
    # Connect to a share
    smbclient //10.10.12.13/notes
    # Download files from SMB
    $ get prod.txt
    # execute local system commands
    $ !cat prod.txt
    #auth connect
    smbclient -L //10.129.202.41 -U alex%lol123!mD 
    ```
    
- `smbstatus` is used to check for connected hosts and which share the client is connected to.
- RPCCLIENT :
    - To interact manually with `smb` we can use `rpcclient` and perform MS-RPC functions and execute them on the SMB server :
        
        ```bash
        rpcclient -U "" 10.129.14.128
        ```
        
        - some RPCCLIENT functions
            
            
            | **Query** | **Description** |
            | --- | --- |
            | `srvinfo` | Server information. |
            | `enumdomains` | Enumerate all domains that are deployed in the network. |
            | `querydominfo` | Provides domain, server, and user information of deployed domains. |
            | `netshareenumall` | Enumerates all available shares. |
            | `netsharegetinfo <share>` | Provides information about a specific share. |
            | `enumdomusers` | Enumerates all domain users. |
            | `queryuser <RID>` | Provides information about a specific user. |
            | `querygroup` | Provides information about a specific group. |
    - To brute force user RIDs :
        
        ```bash
        for i in $(seq 500 1100);do rpcclient -N -U "" 10.129.244.187 -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo "";done
        ```
        
- SMBMAP :
    
    ```bash
    smbmap -H 10.129.14.128
    ```
    
- CrackMapExec :
    
    ```bash
    crackmapexec smb 10.129.14.128 --shares -u '' -p ''
    ```
    
- Enum4Linux :
    
    ```bash
    ./enum4linux-ng.py 10.129.14.128 -A
    ```
    

### NFS (111,2049)

- Same purpose as SMB, but only used in Linux and Unix
- Enumerate NFS through `nmap` :
    
    ```bash
    $ sudo nmap 10.129.14.128 -p111,2049 -sV -sC
    # enumerate through rpcinfo NSE scripts
    $ sudo nmap --script nfs* 10.129.14.128 -sV -p111,2049
    ```
    
- Show available NFS shares :
    
    ```bash
    showmount -e 10.10.12.13
    ```
    
- Mounting NFS share :
    
    ```bash
    $ mkdir target-NFS
    $ mount -t nfs 172.16.139.35:/ ./SRV01/ -o nolock
    $ cd target-NFS
    ```
    
- List content with usernames & group names :
    
    ```bash
    ls -l mnt/nfs/
    ```
    
- List contents with UIDs & GUIDs
    
    ```bash
    ls -n mnt/nfs/
    ```
    
- Unmounting
    
    ```bash
    sudo unmout ./taget-NFS
    ```
    

### DNS

| `PTR` | The PTR record works the other way around (reverse lookup). It converts IP addresses into valid domain names. |
| `SOA` | Provides information about the corresponding DNS zone and email address of the administrative contact. (beginning of the zone file) |

- Checking `soa` info through `dig` :
    
    ```bash
    dig soa www.inlanefreight.local
    ```
    
- Dangerous settings
    
    
    | **Option** | **Description** |
    | --- | --- |
    | `allow-query` | Defines which hosts are allowed to send requests to the DNS server. |
    | `allow-recursion` | Defines which hosts are allowed to send recursive requests to the DNS server. |
    | `allow-transfer` | Defines which hosts are allowed to receive zone transfers from the DNS server. |
    | `zone-statistics` | Collects statistical data of zones. |
- `dig` queries :
    
    ```bash
    # NS Query
    $ dig ns inlanefreight.htb @10.10.11.12
    # Version Query (DNS server)
    $ dig CH TXT version.bind 10.10.12.13
    # View all available entries that the server is willing to disclose
    $ dig any inlanefreight.htb @10.10.12.13
    # ask for the DNS zone contents (test zone transfer)
    $ dig axfr inlanefreight.htb @10.10.12.13
    ```
    
- Subdomain brute force :
    - Dig
        
        ```bash
        for sub in $(cat /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt);do dig $sub.inlanefreight.htb @10.129.14.128 | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt;done
        ```
        
    - DNSENUM
        
        ```bash
        dnsenum --dnsserver 10.129.14.128 --enum -p 0 -s 0 -o subdomains.txt -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt inlanefreight.htb
        ```
        

### SMTP (25,587,465)

- SMTP is often combined with the IMAP or POP3 protocols, which can fetch emails and send emails.


- Connect to the SMTP server :
    
    ```bash
    telnet 10.129.202.41 25
    ```
    
- Interact with the SMTP server : [SMTP error codes](https://serversmtp.com/smtp-error/)
    
    ```bash
    # initiate connection with the server (HELO/EHLO)
    HELO mail1.inlanefreight.htb
    # 252 code confirm the non existence of a user
    VRFY root
    ```
    
- Enumeration using `nmap`
    
    ```bash
    sudo nmap <ip> -sC -sV -p25 # (smtp-commands in default script)
    # to verify if the server is an open relay through 16 tests
    sudo nmap 10.129.14.128 -p25 --script smtp-open-relay -v 
    ```
    

### IMAP / POP3

- By default, ports `110` and `995` are used for POP3, and ports `143` and `993` are used for IMAP. The higher ports (`993` and `995`) use TLS/SSL to encrypt the communication between the client and server.

- IMAP commands :
    
    
    | **Command** | **Description** |
    | --- | --- |
    | `1 LOGIN username password` | User's login. |
    | `1 LIST "" *` | Lists all directories. |
    | `1 CREATE "INBOX"` | Creates a mailbox with a specified name. |
    | `1 DELETE "INBOX"` | Deletes a mailbox. |
    | `1 RENAME "ToRead" "Important"` | Renames a mailbox. |
    | `1 LSUB "" *` | Returns a subset of names from the set of names that the User has declared as being `active` or `subscribed`. |
    | `1 SELECT INBOX` | Selects a mailbox so that messages in the mailbox can be accessed. |
    | `1 UNSELECT INBOX` | Exits the selected mailbox. |
    | `1 FETCH <ID> all` | Retrieves data associated with a message in the mailbox. |
    | `1 CLOSE` | Removes all messages with the `Deleted` flag set. |
    | `1 LOGOUT` | Closes the connection with the IMAP server. |
    | `1 FETCH 1 BODY[]` | Fetch the content of a selected inbox mail |

- POP3 commands :
    
    
    | **Command** | **Description** |
    | --- | --- |
    | `USER username` | Identifies the user. |
    | `PASS password` | Authentication of the user using its password. |
    | `STAT` | Requests the number of saved emails from the server. |
    | `LIST` | Requests from the server the number and size of all emails. |
    | `RETR id` | Requests the server to deliver the requested email by ID. |
    | `DELE id` | Requests the server to delete the requested email by ID. |
    | `CAPA` | Requests the server to display the server capabilities. |
    | `RSET` | Requests the server to reset the transmitted information. |
    | `QUIT` | Closes the connection with the POP3 server. |
- Use IMAP/POP3 on mail server :
    
    ```bash
    # authenticate via curl
    curl -k 'imaps://10.10.12.11' --user user:password123
    # interact with the POP3 server over SSL (we use openssl or nc)
    openssl s_client -connect 10.129.202.20:pop3s
    # interact with the IMAP server over SSL (we use openssl or nc)
    openssl s_client -connect 10.129.202.20:imaps
    ```
    

### SNMP (161,162)

- `Simple Network Management Protocol` ([SNMP](https://datatracker.ietf.org/doc/html/rfc1157)) was created to monitor network devices
- SNMPv1 :
    - Does not support encryption
    - Has no built-in authentication
- SNMPv2 : has no built-in encryption
- Automated footprinting tools :  `snmpwalk`, `onesixtyone`, and `braa`
- Enumeration
    
    ```bash
    # if the community string is known, 'public' in this example
    snmpwalk -v2c -c public 10.129.254.10
    # if we dont know the community string
    onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt 10.129.202.41
    # once we know a community string we can use braa to enumerate info behind every OID
    # syntax : braa <community string>@<IP>:.1.3.6.*
    braa public@10.129.14.128:.1.3.6.*
    ```
    

⇒ Some community strings are bound to specific IPs so they are named after the hostname of the host or sometimes symbols are added to these names.

### MySQL (3306)

- nmap
    
    ```bash
    sudo nmap 10.129.254.10 -sV -sC -p3306 --script mysql*
    ```
    
- mysql
    
    ```bash
    mysql -u robin -probin -h 10.129.254.10
    ```
    

### MSSQL (1433)

- nmap
    
    ```bash
    sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER -sV -p 1433 10.129.230.249
    ```
    
- `Metasploit`
    
    ```bash
    msf6 auxiliary(scanner/mssql/mssql_ping) > set rhosts 10.129.201.248
    ```
    
- Interact with the DB via Impacket `mssqlclient` script
    
    ```bash
    python3 mssqlclient.py backdoor@10.129.230.249 -windows-auth
    ```
    

### Oracle TNS (1521)

- Oracle 9 has a default password :`CHANGE_ON_INSTALL`
- The Oracle DBSNMP service also uses a default password : `dbsnmp`
- Settings and its description in `tnsnames.ora`
    
    
    | **Setting** | **Description** |
    | --- | --- |
    | `DESCRIPTION` | A descriptor that provides a name for the database and its connection type. |
    | `ADDRESS` | The network address of the database, which includes the hostname and port number. |
    | `PROTOCOL` | The network protocol used for communication with the server |
    | `PORT` | The port number used for communication with the server |
    | `CONNECT_DATA` | Specifies the attributes of the connection, such as the service name or SID, protocol, and database instance identifier. |
    | `INSTANCE_NAME` | The name of the database instance the client wants to connect. |
    | `SERVICE_NAME` | The name of the service that the client wants to connect to. |
    | `SERVER` | The type of server used for the database connection, such as dedicated or shared. |
    | `USER` | The username used to authenticate with the database server. |
    | `PASSWORD` | The password used to authenticate with the database server. |
    | `SECURITY` | The type of security for the connection. |
    | `VALIDATE_CERT` | Whether to validate the certificate using SSL/TLS. |
    | `SSL_VERSION` | The version of SSL/TLS to use for the connection. |
    | `CONNECT_TIMEOUT` | The time limit in seconds for the client to establish a connection to the database. |
    | `RECEIVE_TIMEOUT` | The time limit in seconds for the client to receive a response from the database. |
    | `SEND_TIMEOUT` | The time limit in seconds for the client to send a request to the database. |
    | `SQLNET.EXPIRE_TIME` | The time limit in seconds for the client to detect a connection has failed. |
    | `TRACE_LEVEL` | The level of tracing for the database connection. |
    | `TRACE_DIRECTORY` | The directory where the trace files are stored. |
    | `TRACE_FILE_NAME` | The name of the trace file. |
    | `LOG_FILE` | The file where the log information is stored. |
- ODAT: Oracle Database Attacking Tool
    
    ```bash
    ./odat.py all -s 10.129.205.19
    # upload a non-dangerous file for testing
    ./odat.py utlfile -s 10.129.204.235 -d XE -U scott -P tiger --sysdba --putFile C:\\inetpub\\wwwroot testing.txt ./testing.txt
    ```
    
- Enumeration using `nmap`
    
    ```bash
    # general enumeration
    sudo nmap -p1521 -sV 10.129.204.235 --open
    # SID brute forcing
    sudo nmap -p1521 -sV 10.129.204.235 --open --script oracle-sid-brute
    ```
    
- Connecting to a `SQLplus` database after finding valid credentials (all SQLplus commands :  [SQLplus commands](https://docs.oracle.com/cd/E11882_01/server.112/e41085/sqlqraa001.htm#SQLQR985)
    
    ```bash
    # basic connection
    sqlplus scott/tiger@10.129.205.19/XE
    SQL> select table_name from all_tables;
    # in case an error come up in libsqlplus.so, execute this 
    sudo sh -c "echo /usr/lib/oracle/12.2/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf";sudo ldconfig
    # if our user have admin privileges, we can reconnect as sysdba
    sqlplus scott/tiger@10.129.205.19/XE as sysdba
    SQL> select name, password from sys.user$;
    ```
    

### IPMI (623/UDP)

- Enumeration using `nmap` :
    
    ```bash
    sudo nmap -sU --script ipmi-version -p 623 ilo.inlanfreight.local
    ```
    
- `Metasploit` module
    
    ```bash
    msf6 > use auxiliary/scanner/ipmi/ipmi_version 
    ```
    
- Default credentials
    
    
    | **Product** | **Username** | **Password** |
    | --- | --- | --- |
    | Dell iDRAC | root | calvin |
    | HP iLO | Administrator | randomized 8-character string consisting of numbers and uppercase letters |
    | Supermicro IPMI | ADMIN | ADMIN |
- We can dump hashes using the flaw in the RAKP protocol in IPMI 2.0 then crack it offline using `hashcat` :
    1. Dump hashes using `Metasploit`
        
        ```bash
        msf6 > use auxiliary/scanner/ipmi/ipmi_dumphashes 
        ```
        
    2. Hashcat cracking
        
        ```bash
        hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u
        ```
        

### Linux Remote Management Protocols

- Tool to automate **SSH** audit : [https://github.com/jtesta/ssh-audit](https://github.com/jtesta/ssh-audit)
- **Rsync** on port 873 is used for locally and remotely copying files.
- Enumerating shares for Rsync :
    
    ```bash
    # probing accessible shares
    nc -nv 127.0.0.1 873
    # enumerating on open shares
    rsync -av --list-only rsync://127.0.0.1/dev
    ```
    
- **R-Services** are a suite of services hosted to enable remote access or issue commands between Unix hosts over TCP/IP, it span across the ports : `512`, `513`, and `514`
- R-commands :
    - rcp (`remote copy`)
    - rexec (`remote execution`)
    - rlogin (`remote login`)
    - rsh (`remote shell`)
    - rstat
    - ruptime
    - rwho (`remote who`)

### Windows Remote Management Protocols

- NLA : requires the connecting user to authenticate themselves before a session is established with the server
- RDP Enumeration using `nmap` :
    
    ```bash
    # nmap 
    nmap -sV -sC 10.129.201.248 -p3389 --script rdp*
    # nmap scan for hardened networks
    nmap -sV -sC 10.129.201.248 -p3389 --packet-trace --disable-arp-ping -n
    ```
    
- WinRM enumeration using `nmap` :
    
    ```bash
    nmap -sV -sC 10.129.201.248 -p5985,5986 --disable-arp-ping -n
    ```
    
- WMI : for accessing all settings on Windows systems and an extension of the common information model (CIM) core functionality of the Web-Based Enterprise Management.
    - Footprinting : (port 135)
        
        ```bash
        /usr/share/doc/python3-impacket/examples/wmiexec.py Cry0l1t3:"P455w0rD!"@10.129.201.248 "hostname"
        ```

## Virtual Hosts Enumeration

- Using `gobuster`
    ```bash
gobuster vhost -u <target-domain> -w <wordlist>
    ```

- Using `ffuf`
    ```bash
ffuf -w <wordlist> -u <target-ip> -H "Host: FUZZ.<target-domain>"
    ```